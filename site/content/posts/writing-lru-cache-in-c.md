---
title: "Writing LRU Cache in C"
date: 2019-03-13T18:30:56-04:00
draft: false
tags: ["c"]
---



In this post we will discuss the process of creating a Least Recently Used(LRU) cache structure in C. The Least Recently Used policy is very common model to create caching structures in computers.

The [LRU cache eviction policy](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) is as simple as it sounds. It describes the eviction strategy of data in a cache, in this case, if the cache requires to evict data it will evict the least recently used item.

Why and when would we need such a structure?

#### Cache

First of all, let us define a cache. In computer science, a cache(pronounced "cash") is simply a local data store we use to reduce data retrival time. While this concept might seem very simple, it is equally important in all levels of abstraction in a computer. 

For example, in the computer hardware level we can store terabytes of information in a hard drive. However, the speed of the hard drive is slow and cannot compare to the operations a cpu can do within the same timeframe, so we put the running application on the RAM. Now, the cpu can work with data in the RAM instead of the hard drive. In this case, we can say the RAM is a "cache" for the hard drive.

Even in the software level, we have different caches. A cache to store the result of heavy computation, result of network calls, or data retrieved from the database.

By the nature of caching, we can see infer that it exists because we have finite memory at every level of abstraction we work on. Then we know that if we build a cache we should work on the assumption of limited memory available to us. Thus, we have the need for a cache eviction policy. Here, we define eviction as the removal of entries from the cache after it cannot store any more elements.

There are various different cache eviction strategies like First In First Out (FIFO), Least Recently Used (LRU), Most Frequently Used(MFU), or Random. Each one would be appropriate depending on requirements of the system.

#### Datastructure

For our LRU cache, let us define the interface for it and then the internal design specifications that is required.

The cache itself should be very simple, supporting only the basic operations: **get** and **put**.

```c
// Interface for the cache
void* get(char* key);
void put(char* key, void* value);
```

Since the requirement from cache arises to make a current system faster, the time complexity should not be anything else than constant.

We see the cache interface resembles most closely to a hashtable structure found in most of computer science. So our implementation will include a hashtable with slight modifications. With a hashtable we have a basic cache, the ability to store and retrieve data. But to implement our eviction strategy, we need to remember the items that are being read and the items which are not being used. To perform such operations we implement a hashtable with a doubly linked list structure. The linked structure allows us to store data and move them around freely. This enables us to say, keep all the least used items at the tail of the list, while moving the used items to the front of the list. Then, we use hashtable to quickly jump around and get items from the list. Both of the operations can now be performed in constant time.

Implementation of the LRU Cache:

```c
typedef struct Node {
    char* key;
    char* value;
    struct Node* next;
    struct Node* prev;
    struct Node* hashNext;
} Node;

typedef struct Table {
    int capacity;
    Node* *array;
} Table;

typedef struct List {
    int size;
    int capacity;
    Node* head;
    Node* tail;
} List;

typedef struct LRUCache {
    Table* table;
    List* list;
} LRUCache;
```



Now first lets make the methods to allocate space for these structures:

```c
Node* createNode(char* key, char* value) {
    Node* newNode = (Node*) malloc(sizeof(Node));
    newNode -> key = key;
    newNode -> value = value;
    newNode -> next = NULL;
    newNode -> prev = NULL;
    newNode -> hashNext = NULL;
    return newNode;
}

Table* createTable(int capacity) {
    Table* newhash = (Table*) malloc(sizeof(Table));
    newhash -> capacity = capacity;
    newhash -> array = (Node**) malloc(sizeof(Node) * capacity);
    for(size_t i =0 ;i < capacity; i++)
        newhash -> array[i] = NULL;
    return newhash;
}

List* createList(int capacity) {
    List* newList = (List*) malloc(sizeof(List));
    newList -> size = 0;
    newList -> capacity = capacity;
    newList -> head = newList -> tail = NULL;
    return newList;
}

LRUCache* createCache(int size) {
    LRUCache* newCache = (LRUCache*) malloc(sizeof(LRUCache));   
    Table* table = createTable(size * 2);
    List* list = createList(size);
    newCache -> table = table;
    newCache -> list = list;
    return newCache;
}
```



Now the function to add elements to our cache:

```c
void put(LRUCache* cache, char* key, char* value) {
    Node* valNode = createNode(key, value);

    if(addToHash(cache -> table, valNode)) // already in list
        moveToFront(cache, valNode -> key);    
    else
        addToList(cache, valNode); 
}
```



Here we first create a Node so we can pass the same reference to the other functions.

We will call the *addToHash* function that add the entry to the cache. If the key already exists, it will update with new value and return **1**, otherwise it adds to the hashtable and returns **0**.

If the key was already there, we need to move it to the front of list with *moveToFront*. Or if we added new entry to the hashtable we also add it to list with *addToList*.



Now let's write the function definitions:

**addToHash**

```c
int addToHash(Table* table, Node* valNode) {
    size_t hashCode = getHashCode(table, valNode -> key);    
    if(table -> array[hashCode] != NULL) { // check if is in hash

        Node* curr = table->array[hashCode];

        while(curr -> hashNext != NULL) {
            if(strcmp(curr -> key, valNode -> key) == 0) {
                curr -> value = valNode -> value;
                return 1;
            } 
            curr = curr -> hashNext;
        }
        // last node
        if(strcmp(curr -> key, valNode -> key) == 0) {
                curr -> value = valNode -> value;
                return 1;
        } 

        // add after last node
        curr -> hashNext = valNode;
        return 0;
    } else {
        table -> array[hashCode] = valNode;
        return 0;
    }
}
```

**moveToFront**

```c
void moveToFront(LRUCache* cache, char* key) {
    Table* table = cache -> table;
    List* list = cache -> list;

    // return if only element in list
    if(list -> size == 1) 
        return;

    size_t hashCode = getHashCode(table, key);
    Node* curr = table -> array[hashCode];

    // find in hashMap
    while(curr) {
        if(strcmp(curr -> key, key) == 0)
            break;
        curr = curr -> hashNext;
    }
    // return if doesn't exist
    if(curr == NULL)
        return;

    // move curr to latest/tail of list
    if(curr -> prev == NULL) { // if head in list
        curr -> prev = list -> tail;
        list -> head = curr -> next;
        list -> head -> prev = NULL;
        list -> tail -> next = curr;
        list -> tail = curr;
        list -> tail -> next = NULL;
        return;
    }

    if(curr -> next == NULL) // already latest at tail
        return;
    
    // if curr in middle
    curr -> next -> prev = curr -> prev;
    curr -> prev -> next = curr -> next; 
    curr -> next = NULL;
    list -> tail -> next = curr;
    curr -> prev = list -> tail;
    list -> tail = curr;
}

```

**addToList**

```c
void addToList(LRUCache* cache, Node* valNode) {
    List* list = cache -> list;
    if(list -> size == list -> capacity)
        evictCache(cache);
    
    if (list -> head == NULL) {
        list -> head = list -> tail = valNode;
        list -> size = 1;   
        return; 
    }
    
    valNode -> prev = list -> tail;
    list -> tail -> next = valNode;
    list -> tail = valNode;
    list -> size = list -> size + 1;
    return;
}
```



We see that we evict the cache if the size of list reaches the capacity. Lets write that out too.

```c
void evictCache(LRUCache* cache) {
    Table* table = cache -> table;
    List* list = cache -> list;

    Node* entry = list -> head;
    // return if empty
    if(list -> head == NULL)
        return;
    // remove head and tail if only one node
    if(list -> head == list -> tail) {
        list -> head = NULL;
        list -> tail = NULL;
    } else  {
        // remove entry from list with multiple nodes
        list -> head = entry -> next;
        list -> size = list -> size - 1;
        list -> head -> prev = NULL;
    }
        
    // remove from map
    size_t hashCode = getHashCode(table, entry -> key);
    Node** indirect = &table->array[hashCode]; 
    while((*indirect) != entry)
        indirect = &(*indirect)->next;
    *indirect = entry -> next;

    free(entry);
}
```



Another function we keep using is the *getHashCode* which takes a char buffer and returns a index in the table. We will use a simple implementation of a hashing function for this.

```c
static size_t getHashCode(Table* table, const char* source) {    
    if (source == NULL)
        return 0;
    size_t hash = 0;
    while (*source != '\0') {
        char c = *source++;
        int a = c - '0';
        hash = (hash * 10) + a;     
    } 
    return hash % table -> capacity;
}
```

 

Now that we are able to add entries to the cache, lets write the function to retrieve entries.

```c
char* get(LRUCache* cache, char* key) {
    Table* table = cache -> table;
    size_t hashCode = getHashCode(table, key);
    Node* curr =  table -> array[hashCode];
    while(curr) {
        if(strcmp(curr -> key, key) == 0) {
            moveToFront(cache, key);
            return curr -> value;
        }
        curr = curr -> hashNext;
    }
    return NULL;
}
```



And thats it. Lets test this in the main function.



```c
int main() {
    int cacheSize = 3;
    LRUCache* cache = createCache(cacheSize);

    put(cache, "a" , "b");
    put(cache, "c" , "d");
    put(cache, "e" , "f");
    put(cache, "g" , "h");
    put(cache, "c" , "z");
    put(cache, "e" , "y");

    get(cache, "c");

    
    // print the cache contents from lru to mru
    Node* temp = cache -> list -> head;
    for(size_t i = 0; i < cache -> list -> size; i++) {
        printf("%s %s \n", temp -> key, temp -> value);
        temp = temp -> next;
    }
    return 1;
}
```



And this is the output.

```bash
g h 
e y 
c z 
```



You can find the entire source code at [github](https://github.com/adkri/lrucache).