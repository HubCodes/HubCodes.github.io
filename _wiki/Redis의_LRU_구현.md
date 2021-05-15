---
layout  : wiki
title   : Redis의 LRU 구현
summary : Approximated LRU
date    : 2021-05-15 12:24:45 +0900
updated : 2021-05-15 12:24:45 +0900
tag     : coding
toc     : true
public  : true
parent  : [[index]]
latex   : false
---
* TOC
{:toc}

레디스는 인메모리 캐시로 자주 활용된다. 레디스를 활용해 캐싱을 할 때 선택 가능한 메모리 정리 알고리즘 중에는 LRU 가 있는데, 이 LRU를 사용해 캐싱을 하게 되면 레디스가 어떻게 동작하는지 살펴보자.

이 글에서 사용한 Redis 소스코드의 커밋 해시값은 `df4d916007c285d01b11193272419ab228916d8a` 이다.

# Eviction

머신의 메모리 공간은 한정되어 있기 때문에 캐시에 키를 무한정 삽입할 수는 없다. ([예전 레디스](https://redis.io/topics/virtual-memory)라면 디스크를 사용해 할 수 있었을 지 모르겠지만, 디스크 공간도 유한하다는 사실은 둘째 치고 캐시로서 활용할 때 효용성이 있는지는 잘 모르겠다. 실제로 deprecated 되기도 했고)

이 문제를 해결하기 위해서는 더 이상 키를 삽입할 수 없다고 에러를 내거나, 이미 있는 키들 중에 잘 안 쓸 것 같은 키를 지워서 공간을 확보하는 과정이 필요하다. 이 글에서는 "이미 있는 키들 중에 잘 안 쓸 것 같은 키"를 지우는 전략에 대해 살펴보려고 한다.

## LRU

LRU (Least Recently Used) 알고리즘은 널리 사용되는 캐시 공간 확보 알고리즘 중 하나인데, 최근에 가장 *적게* 사용된 캐시 항목, 즉 마지막으로 참조된 시점으로부터 **유휴 시간**이 가장 긴 캐시 항목을 지워서 공간을 확보하는 방식이다.

LRU 알고리즘을 구현하려면 어떻게 해야 할까? 캐시로 사용할 메모리 공간과 (간단히 `Set<T>` 같은 자료구조를 사용할 수 있다) 캐시 항목들에 대하여 사용 시점을 기준으로 *정렬된 큐*를 들고 있으면서 캐시 참조가 일어날 때마다 해당 항목을 큐의 맨 앞으로 꺼내오고, 만약 정렬된 큐가 꽉 찼는데 캐시에 없는 항목이 새로 들어왔다면 정렬된 큐의 가장 뒤쪽에 있는 가장 오래된 키를 날리고 큐 앞에 키를 추가하는 방법으로 구현이 가능하다.

이 방법은 쉽고 잘 작동하지만 한 가지 작은 문제가 있는데, 만약 캐시되는 키가 많다면 키 순서를 전부 관리하고 있는 정렬된 큐도 그만큼 공간을 차지한다는 점이다. doubly linked list로 큐를 구현한다고 치고, 64비트 주소를 사용하는 머신이라면, prev 포인터와 next 포인터를 가지고 있으면서 큐 항목 하나당 최소한 16바이트를 차지하는 셈이다. 만약 캐시 항목이 10만개일 경우 최소한 1.5MB의 추가 공간이 필요하다. (후술하겠지만 redis의 author인 antirez는 이 문제를 최대한 회피하려 했던 것으로 보인다)

### Redis의 LRU 구현

상기한 구현 방법의 공간 효율성 문제를 피하기 위해 레디스는 다소 다른 방법으로 LRU 알고리즘을 구현하고 있다.

구체적으로 살펴보면 `redisObject` 안에 `lru` 라고 하는 24비트 크기의 필드를 두어 객체의 마지막 사용 시각을 초 단위 타임스탬프 값으로 담아둔다. 이렇게 하면 timestamp가 overflow 되기까지는 194일이 걸린다.

```
// In src/server.h:674

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

구조체 레이아웃 상 앞서있던 두 개의 비트필드 4+4=8비트와 `refcount` 중간에 끼워넣기 위해 일부러 24비트를 선택한 것 같다. 이를 통해 적어도 어떤 특정한 환경 (ex: gcc + intel) 에서는 구조체 크기가 전혀 증가하지 않으면서도 필드 하나가 추가되는 이점을 얻을 수 있다. 추가 공간을 몇 MB조차도 사용하지 않게 만드려고 했던 [antirez의 노력](https://github.com/redis/redis/blame/e150ec7d0c83d56f81496084b2e5f119c958ab6f/src/redis.h)이 돋보이는 대목이다. (It was added later, when memory efficiency was a big concern)

실제로 사용 시각을 갱신하는 쪽은 `lookupKey` 함수이다. `lookupKey` 함수는 이름에서 알 수 있듯 키를 조회할 때마다 호출되는 함수이다.

```
// In src/db.c:62

/* Low level key lookup API, not actually called directly from commands
 * implementations that should instead rely on lookupKeyRead(),
 * lookupKeyWrite() and lookupKeyReadWithFlags(). */
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (!hasActiveChildProcess() && !(flags & LOOKUP_NOTOUCH)){
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}
```

핵심은 `val->lru = LRU_CLOCK()` 이다. 다른 예외상황을 제외하고 `LRU_CLOCK()` 이 happy path에서 하는 일은 현재 시간을 초 단위 타임스탬프로 반환하는 것이므로, `lookupKey` 를 통해 어떤 키를 룩업할 때마다 해당 키의 접근시각을 최신으로 갱신하게 된다.

이렇게 해서 접근시각을 갱신했다. 그럼 이제 어떻게 해야 가장 오랫동안 사용되지 않은 키를 선택하여 제거할 수 있을까?

레디스에 처음 LRU가 구현되었던 시점부터 버전 3.0 이전까지는 랜덤하게 선택한 3개의 키 (나중에는 5개의 키로 기본값이 변경되었다고 한다) 중에서 가장 오랫동안 참조되지 않은 키를 삭제하는 방법으로 제거 작업을 수행했다.

이 구현은 3.0 이후 바뀌었는데, 3.0 이후의 레디스는 `eviction pool` 이라고 부르는 작은 연결 리스트(“pool” of good candidates for eviction)를 가지고 있다. 이 리스트는 유휴 시간에 대해 단조증가하는 순서로 정렬되어 있는데, 리스트의 head는 가장 유휴 시간이 짧은 키를, last는 유휴 시간이 가장 긴 키를 나타낸다. 이 pool의 크기는 16으로 하드코딩되어 있다.

```
// In src/evict.c:53

#define EVPOOL_SIZE 16
```

레디스는 해시테이블에서 랜덤하게 키들을 골라낸 다음 풀에 빈 공간이 있거나 풀에 있는 키 중 하나보다 유휴 시간이 길다면 풀에 집어넣고, 풀이 꽉 차면 풀의 맨 뒤에 있는 (즉 가장 오랫동안 쓰이지 않은) 키를 해시테이블과 풀에서 제거한다. 그리고 메모리에 여유가 생길 때까지 이 동작을 반복한다. 즉 일종의 확률적 LRU 알고리즘을 수행한다고 볼 수 있는 것이다.

구체적인 구현은 `evict.c` 파일에 있는 `performEvictions` 함수에서 확인할 수 있는데, 흥미로운 부분 위주로 짚고 넘어가보자.

```
// In src/evict.c:538

while (mem_freed < (long long)mem_tofree) {
```

"메모리에 여유가 생길 때 까지 이 동작을 반복" 에 해당하는 `while` 루프이다. `mem_freed` 변수는 루프를 수행함에 따라 갱신된다.

```
// In src/evict.c:547

if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
            server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
        {
```

LRU라면 이 조건문을 타게 된다.

```
// In src/evict.c:552

while(bestkey == NULL) {
    unsigned long total_keys = 0, keys;

    /* We don't want to make local-db choices when expiring keys,
     * so to start populate the eviction pool sampling keys from
     * every DB. */
    for (i = 0; i < server.dbnum; i++) {
        db = server.db+i;
        dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
                db->dict : db->expires;
        if ((keys = dictSize(dict)) != 0) {
            evictionPoolPopulate(i, dict, db->dict, pool);
            total_keys += keys;
        }
    }
    if (!total_keys) break; /* No keys to evict. */

    /* Go backward from best to worst element to evict. */
    for (k = EVPOOL_SIZE-1; k >= 0; k--) {
        if (pool[k].key == NULL) continue;
        bestdbid = pool[k].dbid;

        if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
            de = dictFind(server.db[pool[k].dbid].dict,
                pool[k].key);
        } else {
            de = dictFind(server.db[pool[k].dbid].expires,
                pool[k].key);
        }

        /* Remove the entry from the pool. */
        if (pool[k].key != pool[k].cached)
            sdsfree(pool[k].key);
        pool[k].key = NULL;
        pool[k].idle = 0;

        /* If the key exists, is our pick. Otherwise it is
         * a ghost and we need to try the next element. */
        if (de) {
            bestkey = dictGetKey(de);
            break;
        } else {
            /* Ghost... Iterate again. */
        }
    }
}
```

내용이 조금 길지만 두 번째 `for` 루프에서 eviction pool을 **뒤에서부터 순회**하는 부분이 중요하다. 앞서 이야기했듯 eviction pool은 단조증가하는 형태로 정렬되어 있기 때문에 뒤에 있는 키일수록 오랫동안 접근되지 않은 키라고 판단할 수 있고, 이를 토대로 `de = dictFind(server.db[pool[k].dbid].dict, pool[k].key);` 에서 해당 키가 실제로 db에 들어 있는지 확인하여 `bestkey` 변수에 해당 키를 집어넣는다. 루프의 탈출 조건이 `bestkey` 가 `NULL` 이 아닌 것이므로, 해당하는 오래된 키를 찾게 되면 이 루프에서 빠져나간다.

LRU 옵션을 확인하는 조건문을 빠져나오면 아래 조건문을 타게 된다. 방금 찾아낸 `bestkey` 가 있으면 실행되는 코드이다.

```
// In src/evict.c:622

if (bestkey) {

    // ... (잘라냄)

    if (server.lazyfree_lazy_eviction)
        dbAsyncDelete(db,keyobj);
    else
        dbSyncDelete(db,keyobj);
```

다른 코드가 많지만 결국 해당하는 키를 가진 항목을 제거하는 내용이 핵심이다. 조건문에 따라 비동기 삭제와 동기 삭제가 나뉘고 있는데, 레디스 파라미터로 `lazyfree-lazy-eviction` 이 설정되어 있는지 여부에 따라 다를 뿐 삭제는 수행하게 된다. 이렇게 삭제한 다음 `mem_freed` 변수를 갱신한다.

### 성능

[https://redis.io/topics/lru-cache#approximated-lru-algorithm](https://redis.io/topics/lru-cache#approximated-lru-algorithm) 에서 볼 수 있듯이, 이렇게 구현된 LRU 알고리즘은 샘플 수만 충분하게 주어진다면 theorical한 구현체에 꽤 근접하는 정확도를 보여 주고 있다. 만약 샘플 수를 조정하고 싶다면 `CONFIG SET maxmemory-samples <count>` 으로 간단하게 할 수 있다고 하니, 실 운영 환경에서는 캐시 히트율과 CPU 사용률 같은 지표를 모니터링 하면서 적절한 샘플 값을 찾아 튜닝하면 좋을 것 같다.

## References

- [https://github.com/redis/redis](https://github.com/redis/redis)
- [https://redis.io/topics/lru-cache](https://redis.io/topics/lru-cache)
- [http://antirez.com/news/109](http://antirez.com/news/109)
- [https://tokers.github.io/posts/lru-and-lfu-in-redis-memory-eviction/](https://tokers.github.io/posts/lru-and-lfu-in-redis-memory-eviction/)
