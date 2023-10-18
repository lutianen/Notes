# 秋招刷题笔记

## 易混辨析

- 子串一定是连续的，而子序列不一定是连续的
- 组合：不强调元素顺序 $C(n, k) = n! / k!(n - k)!$
- 排列：强调元素顺序 $A(n, k) = n! / (n - k)!$

## 常用技巧

### `0x3F3F3F3F`

  使用 `0x3F3F3F3F` 代替最大值 `std::INT_MAX`，防止运算过程中数值溢出

### 迭代器二分

  ```cpp
  std::vector<int> nums{1, 2, 3, 44, 54, 96, 124};
  auto k = std::lower_bound(nums.begin(), nums.end(), 56) - nums.begin();  // k = 5
  k = std::upper_bound(nums.begin(), nums.end(), 36) - nums.begin();  // k = 3
  ```

### 优先队列 - 小根堆、大根堆

  ```cpp
  std::priority_queue<int> pq; // 默认大根堆
  std::priroity_queue<int, vector<int>, std::greater<int>> pq; // 小根堆
  ```

### 字符串分割

  利用字符流，将旧串`s`塞入流`ss`中，然后读入到新的字符串 `str` 中

   ```cpp
   std::string s = "hello, this is Lute~";
   std::stringstream ss(s);
   std::string str;
   std::vector<std::string> vec;
   while (ss >> str) vec.push_back(str);
   
   std::cout << vec.size() << std::endl;
   for (const auto& str : vec) {
       std::cout << str << "\n";
   } 
   ```

### 四舍五入保留小数

  ```cpp
  char str[10];
  double num = 3.1415926;
  sprintf(str, "%.2f", num);
  string s = str;
  cout << s << endl;  // 3.14
  ```

### 字符串按格式拆分

   ```cpp
   std::string str = "12:34@23";
   int u, v, w;
   sscanf(str.c_str(), "%d:%d@%d", &u, &v, &w);
   std::cout << "u = " << u << ", v = " << v << ", w = " << w << std::endl; // u = 12, v = 34, w = 23
   ```

## 正则表达式

  ```cpp
  #include <algorithm>
  #include <iostream>
  #include <regex>
  #include <sstream>
  #include <string>
  using namespace std;

  int main() {
      std::cout << __FUNCTION__ << std::endl;

      // 目标串
      string str = "nums1=[12,23,4,9],nums2=[1,2,3],k=12";
      // 匹配模式
      regex r("(nums1=\\[)([0-9,]+)(\\],nums2=\\[)([0-9,]+)(\\],k=)(\\d+)");

      // 匹配结果
      string s1, s2;
      int k;

      for (sregex_iterator it(str.begin(), str.end(), r), end; it != end; ++it) {
          smatch match = *it;
          // for (size_t i = 0; i < match.size(); ++i) {
          //     cout << "i = " << i << ": " << match[i] << endl;
          // }
          // cout << match[2] << endl;
          s1 = match[2];
          // cout << match[4] << endl;
          s2 = match[4];
          // cout << match[6] << endl;
          k = stoi(match[6]);
      }

      // 根据 ',' 分割字符
      istringstream iss1(s1);
      istringstream iss2(s2);
      vector<int> nums1, nums2;
      string temp;
      while (getline(iss1, temp, ',')) nums1.push_back(stoi(temp));
      while (getline(iss2, temp, ',')) nums2.push_back(stoi(temp));

      // 验证结果
      for (const auto& num : nums1) cout << num << ", ";
      cout << endl;
      for (const auto& num : nums2) cout << num << ", ";
      cout << endl;
      cout << k << "\n";

      return 0;
  }
  ```

## 多线程

### `std::thread` 与 `pthread` 创建线程消耗对比

C++ 标准库中的 thread 库

```cpp
#include <benchmark/benchmark.h>
#include <thread>

void BM_for(benchmark::State& bm) {
    for (auto _:bm) {
        std::thread t1([](){
        });
        t1.join();
    }
}
BENCHMARK(BM_for);
BENCHMARK_MAIN();

/*
    2023-09-08T14:40:17+08:00
    Run on (16 X 4600 MHz CPU s)
    CPU Caches:
    L1 Data 48 KiB (x8)
    L1 Instruction 32 KiB (x8)
    L2 Unified 1280 KiB (x8)
    L3 Unified 24576 KiB (x1)
    Load Average: 3.26, 2.91, 2.36
    ***WARNING*** CPU scaling is enabled, the benchmark real time measurements may be noisy and will incur extra overhead.
    -----------------------------------------------------
    Benchmark           Time             CPU   Iterations
    -----------------------------------------------------
    BM_for          14138 ns         8664 ns        71463

    14138 ns -->> 14.138 us
*/
```

Linux 中的 pthread 库

```cpp
#include <benchmark/benchmark.h>
#include <pthread.h>

void* threadFunc (void*) { return nullptr; }

void BM_for(benchmark::State& bm) {
    for (auto _:bm) {
        pthread_t pid;
        pthread_create(&pid, nullptr,
            threadFunc, nullptr);

        pthread_join(pid, nullptr);
    }
}
BENCHMARK(BM_for);
BENCHMARK_MAIN();

/*
    2023-09-08T14:45:26+08:00
    Run on (16 X 4600 MHz CPU s)
    CPU Caches:
    L1 Data 48 KiB (x8)
    L1 Instruction 32 KiB (x8)
    L2 Unified 1280 KiB (x8)
    L3 Unified 24576 KiB (x1)
    Load Average: 2.63, 2.55, 2.34
    ***WARNING*** CPU scaling is enabled, the benchmark real time measurements may be noisy and will incur extra overhead.
    -----------------------------------------------------
    Benchmark           Time             CPU   Iterations
    -----------------------------------------------------
    BM_for          13346 ns         8402 ns        72869

    13346 ns -->> 13.346 us
*/
```

### `std::async`

特点

- `std::async` 接受一个带返回值的 Lambda, 自身返回一个 `std::future` 对象
- Lambda 可以返回 `void`, 此时`std::future` 对象类型为 `std::future<void>`
- Lambda 在另一个线程中执行, 最后调用 `future` 对象的 `get` 函数（如果此时 Lambda 未执行完成，则会等待）获取返回值
- **`std::future` 删除了拷贝构造函数、拷贝赋值函数**, 因此如果需要浅拷贝，实现共享同一个 `future` 对象, 则改用 `std::shared_future`

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <thread>

void download(const string& str) {
    for (size_t i = 0; i <= 10; ++i)
        printf("%s downloading ... %zu\% \r\n", str.c_str(), i * 10);
    printf("%s download finished. \r\n", str.c_str(), i * 10);

    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}

int main() {
    std::future<int> ftr = std::async([&](){
        download("Lute.tar.gz");
        return 12;
    });
    int rc = ftr.get();
    std::cout << "rc = " << rc << std::endl;
    return 0;
}
```

### `std::mutex` 互斥锁

互斥量，防止多个线程同时进入某一段代码：`lock()` / `unlock()`

`std::lock_guard` 符合 RAII 思想的上锁和解锁：`std::lock_guard lock(mtx);`

- 在构造函数中调用 `mtx.lock();`, 上锁
- 在析构函数中调用 `mtx.unlock();`, 在离开作用域时自动解锁
- **`std::lock_guard` 无非是调用其构造参数为 `lock()` 的成员函数，因此 `std::unique_lock()` 也可以作为 `std::lock_guard` 的构造参数**

    ```cpp
    mutex mtx;
    unique_lock<mutex> u_lock(mtx);
    lock_guard<unique_lock> lock(u_lock);
    ```

    ---

`std::unique_lock`, 不仅仅符合 RAII 思想，而且自由度更高（可以提前释放 `mtx.unlock()`）：*`std::unique_lock<mutex> lock(mtx)` 在析构函数调用时，会检测额外的 FLAG 标志（标志 `mtx` 是否已被释放），然后在决定是否调用 `mtx.unlock();`*

`std::unique_lock` 具有 `mutex` 的所有成员函数：`lock()` / `try_lock()` / `try_lock_for()` 等，且还会在析构时自动调用 `unlock()`

> 概念 concept（鸭子类型）：只要具有某些特定名字的成员函数，就判断一个类是否满足某些功能的思想，在 C++ 中称为 concept（概念）， 而在 Python 中成为鸭子类型
>
> 比起虚函数和动态多态的接口抽象，concept 使实现和接口更加解耦且没有性能损失

### `std::condition_variable` 条件变量

**必须和`std::unique_lock<mutex> lock(mtx)` 一起使用**

- `cv.wait(lock);` 将会让当前线程陷入等待，该函数还可以额外指定一个参数`expr`（`expr` 是一个 Lambda，只有其返回值为 `true` 时才会真正唤醒，否则继续等待）, 即`cv.wait(lock, expr)`
- `cv.notify_one();` 唤醒某个陷入等待的线程；
- `cv.notify_all();` 唤醒所有陷入等待的线程；

### `n` 个线程交替打印本线程 ID(0 ~ n - 1), 重复打印 `m` 次

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

std::atomic_int g_cnt {0};
std::atomic_int g_m{0};
std::vector<std::thread> g_threads;

int main() {
    int n = 4, m = 2;
    g_m.store(m);

    for (int i = 0; i < n; ++i) {
        g_threads.push_back(
            std::move(
                std::thread (
                    [&](int thread_id){
                        while (g_m.load() > 0) {
                            if (g_cnt.load() == thread_id) {
                                std::cout << thread_id << std::endl;
                                g_cnt.fetch_add(1);
                                if (g_cnt.load() == n) {
                                    g_m.fetch_sub(1);
                                    g_cnt.store(0);
                                }
                            }
                        }
                    }, i
                )
            )
        );
    }

    for (auto& thread : g_threads) thread.join();
    return 0;
}
```

### 生产者 - 消费者模式

```cpp
template <typename T>
class MTQuque {
public:
    void push(T val) {
        std::unique_lock<mutex> lock(mtx_);
        vec_.push_back(std::move(val));
        cv_.notify_one();
    }

    T pop() {
        std::unique_lock<mutex> lock(mtx_);
        cv_.wait(lock, [this] () { return !vec_.empty(); });
        T temp = std::move(vec_.back());
        vec_.pop_back();
        return temp;
    }

    void pushMany(std::initializer_list<T> vals) {
        std::unique_lock<mutex> lock(mtx_);
        std::copy(std::move_iterator<decltype(vals.begin())>(vals.begin()), 
                    std::move_iterator<decltype(vals.end())>(valid.end()), 
                    std::back_iterator<std::vector<T>>(vec_));
        cv_.notify_all();
    }

private:
    std::mutex mtx_;
    std::condition_variable cv_;
    std::vector<T> vec_;
};

int main() {
    MTQuque<int> foods;
    std::thread t1([&](){
        for (int i = 0; i < 2; ++i) {
            auto food = foods.pop();
            std::cout << food << std::endl;
        }
    });

    std::thread t2([&] () {
        for (int i = 0; i < 2; ++i) {
            auto food = foods.pop();
            std::cout << food << std::endl;
        }
    });

    foods.push(12);
    foods.push(13);

    foods.pushMany({14, 15});

    t1.join();
    t2.join();
    return 0;
}
```

## 智能指针

**unique_ptr**: 底层是一个`tuple`，存储的是`指针`和对应的`Deleter`

**shared_ptr**: 底层是两根指针，一根指向资源，一根指向原子变量的引用计数；

![unique_ptr](https://raw.githubusercontent.com/lutianen/PicBed/master/smart_pointer.svg)

    ---

## Array 数组

### [数组中重复的数字](https://www.nowcoder.com/practice/6fe361ede7e54db1b84adc81d09d8524?tpId=265&tqId=39207&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

在一个长度为n的数组里的所有数字都在0到n-1的范围内

数组中某些数字是重复的，但不知道有几个数字是重复的。也不知道每个数字重复几次。请找出数组中任意一个重复的数字。

```cpp
int duplicate(vector<int>& nums) {
    unordered_set<int> uset;

    for (const auto& num : nums) {
        if (uset.find(num) == uset.end()) {
            uset.insert(num);
        } else 
            return num;
    }
    return -1;
}
```

```go
func duplicate( numbers []int ) int {
    var uset = map[int]int{} // 类似于 C++ 中的 unordered_set

    for i := 0; i < len(numbers); i++ {
        if _, ok := uset[numbers[i]]; ok { // 判断 numbers[i] 是否在 uset 容器中
            return numbers[i]
        } else {
            uset[numbers[i]] = numbers[i]
        }
    }
    return -1
}
```

### [二维数组中的查找](https://www.nowcoder.com/practice/abc3fe2ce8e146608e868a70efebf62e?tpId=265&tqId=39208&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

在一个二维数组array中（每个一维数组的长度相同），每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。

请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

```cpp
bool Find(int target, vector<vector<int> >& array) {
    int i = array.size() - 1, j = 0;
    
    while (i >= 0 && i < array.size() && j >= 0 && j < array[0].size()) {
        if (array[i][j] < target) {
            ++j;
        } else if (array[i][j] > target) {
            --i;
        } else 
            return true;
    }
    return false;
}
```

```go
func Find(target int, array [][]int) bool {
    i, j := len(array) - 1, 0

    for i >= 0 && i < len(array) && j >= 0 && j < len(array[0]) {
        if array[i][j]  < target {
            j++
        } else if array[i][j] > target {
            i--
        } else { 
            return true
        }
    }

    return false
}
```

## List 链表

### List 通用操作

  ```cpp
  // Singly-linked list
  struct ListNode {
    int val;
    ListNode* next;

    ListNode() : val(0), next(nullptr) {}
    ListNode(int v) : val(v), next(nullptr) {}
    ListNode(int v, ListNode* nxt) : val(v), next(nxt) {}
  };

  ListNode* createList(vector<int>& list) {
    if (list.empty()) return nullptr;

    ListNode* head = new ListNode(list[0]);
    ListNode* curr = head;
    for(size_t i = 1; i < list.size(); ++i) {
      curr->next = new ListNode(list[i]);
      curr = curr->next;
    }

    return head;
  }

  void deleteList(ListNode* head) {
    if (head == nullptr) return;
    if (head->next != nullptr) deleteList(head->next);
    delete head;
  }

  void displayList(ListNode* head) {
    ListNode* curr = head;
    while(curr != nullptr) {
      cout << curr->val;
      if (curr->next != nullptr) cout << " ->";
      curr = curr->next;
    }
  }
  ```

### [从尾到头打印链表](https://www.nowcoder.com/practice/d0267f7f55b3412ba93bd35cfa8e8035?tpId=265&tqId=39210&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

输入一个链表的头节点，按链表从尾到头的顺序返回每个节点的值（用数组返回）

```cpp
vector<int> printListFromTailToHead(ListNode* head) {
    ListNode* curr = head;
    vector<int> res;

    while (curr != nullptr) {
        res.push_back(curr->val);
        curr = curr->next;
    }
    reverse(res.begin(), res.end());
    return res;
}
```

```go
type ListNode struct {
    Val int
    Next *ListNode
}

func printListFromTailToHead(head *ListNode) []int{
    var res []int
    if head == nil {
        return res
    }

    curr := head
    for curr != nil {
        res = append(res, curr.Val)
        curr = curr.Next
    }

    reverse(res)
    return res
}

func reverse(array []int) {
    n := len(array)

    for i := 0; i < n >> 1; i++ {
        swap(&array[i], &array[n - 1 - i])
    }
}

func swap(a *int, b *int) {
    var temp int
    temp = *a
    *a = *b
    *b = temp
}
```

### [合并 K 个升序链表](https://leetcode.cn/problems/merge-k-sorted-lists/)

  **初始把所有链表的头节点入堆，然后不断弹出堆中最小节点 `x`，如果 `x != nullptr && x->next != nullptr`就加入堆中。循环直到堆为空，并把弹出的节点按顺序拼接起来**

  ```cpp
  ListNode* mergeKLists(vector<ListNode*>& lists) {
    if (lists.empty()) return nullptr;

    auto cmp = [](ListNode* pa, ListNode* pb) { return pa->val > pb->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pque(cmp);

    for (auto list : lists)
      if(list != nullptr) pque.push(list);

    ListNode* dummy = new ListNode(-1), *curr = dummy;
    while(!pque.empty()) {
      ListNode* node = pque.top();
      pque.pop();

      curr->next = node;
      curr = curr->next;

      if (node != nullptr && node->next != nullptr)
        pque.push(node->next);
    }

    return dummy->next;
  }
  ```

  ```go
  import "container/heap"

  type ListNode struct {
    Val  int
    Next *ListNode
  }

  func mergeKLists(lists []*ListNode) *ListNode {
    h := hp{}
    for _, head := range lists {
      if head != nil {
        h = append(h, head)
      }
    }
    heap.Init(&h)

    dummy := &ListNode{}
    curr := dummy

    for len(h) > 0 {
      node := heap.Pop(&h).(*ListNode)
      if node.Next != nil {
        heap.Push(&h, node.Next)
      }

      curr.Next = node
      curr = curr.Next
    }

    return dummy.Next
  }

  type hp []*ListNode

  func (h hp) Len() int {
    return len(h)
  }
  func (h hp) Less(i, j int) bool {
    return h[i].Val < h[j].Val
  }
  func (h hp) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
  }
  func (h *hp) Push(v any) {
    *h = append(*h, v.(*ListNode))
  }
  func (h *hp) Pop() any {
    a := *h
    v := a[len(a)-1]
    *h = a[:len(a)-1]
    return v
  }
  ```

### [反转链表](https://leetcode.cn/problems/reverse-linked-list)

  给你单链表的头节点 `head`，请你反转链表，并返回反转后的链表

  ```cpp
  ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr, *curr = head;
    while(curr != nullptr) {
      ListNode* next = curr->next;
      curr->next = prev;
      prev = curr;
      curr = next;
    }
    return prev;
  }
  ```

  ```go
  func reverseList(head *ListNode) *ListNode {
    var prev, curr *ListNode = nil, head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
  }
  ```

### [反转链表 II](https://leetcode.cn/problems/reverse-linked-list-ii/)

给你单链表的头指针 `head` 和两个整数 `left` 和 `right` ，其中 `left <= right`。请你反转从位置 `left` 到位置 `right` 的链表节点，返回 反转后的链表 。

  ```cpp
  ListNode* reverse(ListNode* st, ListNode* ed) {
      ListNode* prev = nullptr;
      ListNode* curr = st;

      while (curr != ed) {
          ListNode* next = curr->next;
          curr->next = prev;
          prev = curr;
          curr = next;
      }
      return prev;
  }

  ListNode* reverseBetween(ListNode* head, int left, int right) {
      ListNode *dummy = new ListNode(-1, head), *prev = dummy;
      for (int i = 0; i < left - 1; ++i) prev = prev->next;

      ListNode *st = prev->next, *curr = prev;
      for (int i = 0; i < right - left + 1; ++i) curr = curr->next;
      ListNode* ed = curr->next;

      ListNode* new_head = reverse(st, ed);
      prev->next->next = ed;
      prev->next = new_head;

      return dummy->next;
  }
  ```

### [K 个一组翻转链表](https://leetcode.cn/problems/reverse-nodes-in-k-group)

给你链表的头节点 head ，每 k 个节点一组进行翻转，请你返回修改后的链表

```cpp
ListNode *reverse(ListNode *a, ListNode *b) {
    ListNode *prev = nullptr, *curr = a;
    while(curr != b) {
        ListNode *next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

ListNode* reverseKGroup(ListNode* head, int k) {
    if (head == nullptr) return head;

    ListNode *a = head, *b = head;
    for (int i = 0; i < k; ++i) {
      if (b == nullptr) return head;
      b = b->next;
    }
    
    ListNode *new_head = reverse(a, b);
    a->next = reverseKGroup(b, k); 

    return new_head;
}
```

### [回文链表](https://leetcode.cn/problems/aMhZSa)

给定一个链表的 头节点 head ，请判断其是否为回文链表

```cpp
ListNode *reverse(ListNode *head) {
  ListNode *prev = nullptr, *curr = head;
  while(curr != nullptr) {
    ListNode *next = curr->next;
    curr->next = prev;
    prev = curr;
    curr = next;
  }
  return prev;
}

bool isPalindrome(ListNode* head) {
  if (head == nullptr || head->next == nullptr) return true;

  ListNode *fast = head, *slow = head;
  while(fast != nullptr && fast->next != nullptr) {
    fast = fast->next->next;
    slow = slow->next;
  }

  // 偶数 -> slow is middle
  if (fast == nullptr)
    slow = reverse(slow);
  else 
    slow = reverse(slow->next);

  fast = head;
  while(slow != nullptr) {
    if (fast->val != slow->val) return false;

    fast = fast->next;
    slow = slow->next;
  }
  return true;
}
```

### [删除排序链表中的重复元素](https://leetcode.cn/problems/remove-duplicates-from-sorted-list)

```cpp
ListNode *deleteDuplicates(ListNode* head) {
    if (head == nullptr) return head;

    ListNode *slow = head, *fast = head->next;
    while(fast != nullptr) {
        if (slow->val != fast->val) {
            slow = slow->next;
            slow->val = fast->val;
        }
        fast = fast->next;
    }

    // TODO delete
    slow->next = nullptr;
    return head;
}
```

    ---

## 字符串

### [相邻字符串不相等的最少替换次数](https://www.geeksforgeeks.org/minimum-replacements-in-a-string-to-make-adjacent-characters-unequal/)

统计使得字符串中相邻字符不同的最小操作次数

```cpp
size_t countMinimun(const string& str) {
    int cnt = 0;
    int i = 0;
    while(i < str.size()) {
        size_t j = i;

        // 查找相邻且相同字符构成的字符串
        while(str[j] == str[i] && j < str.size()) ++j;
        
        // 相同字符串的长度
        size_t diff = j - i;
        cnt += diff / 2;
        i = j;
    }
    return cnt;
}
```

### [替换空格](https://www.nowcoder.com/practice/0e26e5551f2b489b9f58bc83aa4b6c68?tpId=265&tqId=39209&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

```cpp
string replaceSpace(string s) {
    string res;
    for (const auto& ch : s) {
        if (ch == ' ')
            res += "%20";
        else
            res.push_back(ch);
    }
    return res;
}
```

```go
func replaceSpace(s string) string {
  var ret string

  for i := 0; i < len(s); i++ {
    if s[i] == ' ' {
      ret += "%20"
    } else {
      ret += string(s[i])
    }
  }

  return ret
}
```

## Tree 树

### 通用操作

```cpp
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;

    TreeNode () : val(0), left(nullptr), right(nullptr) {}
    TreeNode (int v) : val(v), left(nullptr), right(nullptr) {}
    TreeNode (int v, TreeNode *l, TreeNode *r) : val(v), left(l), right(r) {}
};
```

### [重建二叉树](https://www.nowcoder.com/practice/8a19cbe657394eeaac2f6ea9b0f6fcf6?tpId=265&tqId=39211&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

给定节点数为 n 的二叉树的前序遍历和中序遍历结果，请重建出该二叉树并返回它的头结点

- vin.length == pre.length
- pre 和 vin 均无重复元素
- vin出现的元素均出现在 pre里
- 只需要返回根结点

```cpp
TreeNode* reConstructBinaryTree(vector<int>& preOrder, vector<int>& vinOrder) {
    if (preOrder.empty() || vinOrder.empty()) return nullptr;
    
    int root_val = preOrder[0];
    size_t idx = 0;
    for (; idx < vinOrder.size(); ++idx) {
        if (vinOrder[idx] == root_val) break;
    }

    vector<int> vinOrder_left(vinOrder.begin(), vinOrder.begin() + idx);
    vector<int> vinOrder_right(vinOrder.begin() + idx + 1, vinOrder.end());

    vector<int> preOrder_left(preOrder.begin() + 1,
                              preOrder.begin() + 1 + vinOrder_left.size());
    vector<int> preOrder_right(preOrder.begin() + 1 + vinOrder_left.size(), preOrder.end());

    TreeNode* root = new TreeNode(root_val);

    root->left = reConstructBinaryTree(preOrder_left, vinOrder_left);
    root->right = reConstructBinaryTree(preOrder_right, vinOrder_right);

    return root;
}
```

```go
type TreeNode struct {
    Val int
    Left *TreeNode
    Right *TreeNode
}

func reConstructBinaryTree(preOrder []int, vinOrder []int) *TreeNode {
    if len(preOrder) == 0 || len(vinOrder) == 0 {
      return nil
    }

    root_val, idx := preOrder[0], 0
    for idx < len(vinOrder) {
      if vinOrder[idx] == root_val {
        break
      }
      idx++
    }

    vinOrder_left, vinOrder_right := vinOrder[ : idx], vinOrder[idx + 1 : ]
    fmt.Println(len(vinOrder_left), len(vinOrder_right))

    preOrder_left, preOrder_right := preOrder[1 : len(vinOrder_left) + 1], preOrder[len(vinOrder_left) + 1:]
    fmt.Println(len(preOrder_left), len(preOrder_right))

    root := new(TreeNode)
    root.Val = root_val
    root.Left = reConstructBinaryTree(preOrder_left, vinOrder_left)
    root.Right = reConstructBinaryTree(preOrder_right, vinOrder_right)

    return root
}
```

### [二叉树的下一个节点](https://www.nowcoder.com/practice/9023a0c988684a53960365b889ceaf5e?tpId=265&tqId=39212&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

给定一个二叉树其中的一个结点，请找出中序遍历顺序的下一个结点并且返回

注意，树中的结点不仅包含左右子结点，同时包含指向父结点的next指针

```cpp
struct TreeLinkNode {
    int val;
    TreeLinkNode *left;
    TreeLinkNode *right;
    TreeLinkNode *next;

    TreeLinkNode (int x) : val(x), left(nullptr), right(nullptr) {}
};

TreeLinkNode* GetNext(TreeLinkNode* pNode) {
    function <TreeLinkNode*(TreeLinkNode*)> getParent = [&] (TreeLinkNode * curr) {
        if (curr->next == nullptr)
            return curr;
        else
            return getParent(curr->next);
    };

    TreeLinkNode* root = getParent(pNode);
    vector<TreeLinkNode*> vin_vec{};

    function<void(TreeLinkNode*)> vinOrder = [&] (TreeLinkNode * root) {
        if (root == nullptr)
            return;

        vinOrder(root->left);
        vin_vec.push_back(root);
        vinOrder(root->right);
    };
    vinOrder(root);

    TreeLinkNode* rc = nullptr;
    for (size_t i = 0; i < vin_vec.size(); ++i) {
        if (vin_vec[i] == pNode) {
            rc = vin_vec[i + 1];
            break;
        }
    }

    return rc;
}
```

```go
type TreeLinkNode struct {
    Val   int
    Left  *TreeLinkNode
    Right *TreeLinkNode
    Next  *TreeLinkNode
}

func getParent(curr *TreeLinkNode) *TreeLinkNode {
    if curr == nil {
        return nil
    }

    if curr.Next == nil {
        return curr
    } else {
        return getParent(curr.Next)
    }
}

func GetNext(pNode *TreeLinkNode) *TreeLinkNode {
    if pNode == nil {
        return pNode
    }

    root := getParent(pNode)
    var vec []*TreeLinkNode

    var inOrder func(*TreeLinkNode)
    inOrder = func(root *TreeLinkNode) {
        if root == nil {
            return
        }

        inOrder(root.Left)
        vec = append(vec, root)
        inOrder(root.Right)
    }

    inOrder(root)

    for idx := 0; idx < len(vec); idx++ {
        if vec[idx] == pNode {
            if idx == len(vec)-1 {
                return nil
            }
            return vec[idx+1]
        }
    }

 return nil
}
```

## 单调队列

```cpp
/**
 * 单调队列
 */
class MonitonicQueue {
public:
    void push(int val) {
    // 剔除较小值，保持队列单调性
        while(!que_.empty() && que_.back() < val)
            que_.pop_back();
        que_.push(val);
    }

    int max() const { return que_.front(); }

    void pop(int val) {
        if (val == que_.front())
            que_.pop_front();
    }

private:
    deque<int> que_;
};

```

### [滑动窗口最大值](https://leetcode.cn/problems/sliding-window-maximum/)

给你一个整数数组`nums`，有一个大小为 `k` 的滑动窗口从数组的最左侧移动到数组的最右侧

你只可以看到在滑动窗口内的 `k` 个数字, 滑动窗口每次只向右移动一位。

返回滑动窗口中的最大值

```cpp
vector<int> maxSildingWindow(vector<int>& nums, int k) {
    vector<int> res;
    deque<int> que;

    for (int i = 0; i < nums.size(); ++i) {
        if (i < k - 1) {
            while(!que.empty() && nums[i] > nums[que.back()]) 
                que.pop_back();
            que.push_back(i);
        } else {
            while(!que.empty() && nums[i] > nums[que.back()]) 
                que.pop_back();
            que.push_back(i);
            
            res.push_back(nums[que.front()]);
            
            if (nums[i] == que.front()) que.pop_front();
        }
    }

    return res;
}
```

## Stack 栈

[用两个栈实现队列](https://www.nowcoder.com/practice/54275ddae22f475981afa2244dd448c6?tpId=265&tqId=39213&rp=1&ru=/exam/oj/ta&qru=/exam/oj/ta&sourceUrl=%2Fexam%2Foj%2Fta%3FtpId%3D13&difficulty=undefined&judgeStatus=undefined&tags=&title=)

用两个栈来实现一个队列，使用n个元素来完成 n 次在队列尾部插入整数(push)和n次在队列头部删除整数(pop)的功能

队列中的元素为int类型

保证操作合法，即保证pop操作时队列内已有元素

```cpp

```

### [下一个更大元素](https://leetcode.cn/problems/next-greater-element-i/)

```cpp
vector<int> nextGreaterElement(vector<int>& nums) {
    size_t n = nums.size();
    vector<int> res(n);
    stack<int> stk;

    // 倒着往栈里放
    for (size_t i = n - 1; i >= 0; --i) {
        // 判定个子高矮
        while (!stk.empty() && stk.top() <= nums[i]) {
            // 矮个起开，反正也被挡着了
            stk.pop();
        }
        // nums[i] 身后的更大元素
        res[i] = stk.empty() ? -1 : stk.top();
        stk.push(nums[i]);
    }
    return res;
}
```

### 环形数组

- **通过 `%` 运算符求模（余数），来模拟环形特效**
- **数组长度翻倍**

```cpp
vector<int> nextGreaterElements(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n);
    stack<int> stk;

    // 倒着往栈里放
    for (size_t i = 2 * n - 1; i >= 0; --i) {
        // 判定个子高矮
        while (!stk.empty() && stk.top() <= nums[i % n]) {
            // 矮个起开，反正也被挡着了
            stk.pop();
        }
        // nums[i] 身后的更大元素
        res[i % n] = stk.empty() ? -1 : stk.top();
        stk.push(nums[i % n]);
    }
    return res;
}
```

## 滑动窗口

```cpp
int lengthOfLongestSubstring(string s) {
    unordered_set<char> uset;

    int left = 0, right = 0, length = 0;
    while(right < s.size()) {
        while(uset.find(s[right]) != uset.end()) {
            uset.erase(s[left++]);
        }

        if (right - left + 1 > length) length = right - left + 1;
        uset.insert(s[right++]);
    }

    return length;
}
```

window[left, right) = nums[left, ..., right)， 属于左闭右开的区间

使用**窗口模板**需要考虑以下问题：

- 什么时候应该移动 `right` 扩大窗口？窗口加入字符时，应该更新哪些数据？
- 什么时候窗口应该暂停扩大？移动 `left` 缩小窗口，即从窗口移出字符时，应该更新哪些数据？
- 需要的结果，应该在扩大窗口时还是缩小窗口时进行更形？

```cpp
void slidingWindow(string& str) {
    unordered_map<char, int> window;
    
    size_t left = 0, right = 0;
    while(right < str.size()) {
        char ch = str[right++];
        ++window[ch];

        /* 进行窗口更新 */

        /* Debug 位置 */
        // cout << "window: [" << left << ", " << right << "]" << endl;

        while(left < right && window needs shrink) {
            char d = str[left++];
            window[d]--;

            /* 进行窗口更新 */
        }
    }
}
```

### [最小覆盖字串](https://leetcode.cn/problems/minimum-window-substring)

给你一个字符串`s` 、一个字符串`t` 。返回 `s` 中涵盖 `t` 所有字符的最小子串。如果 `s` 中不存在涵盖 `t` 所有字符的子串，则返回空字符串 ""

```cpp
string minWindow(string str, string tar) {
    unordered_map<char, int> need, window;
    for (auto ch : tar) need[ch]++;

    size_t left = 0, right = 0, valid = 0, start = 0, len = 0x3F3F3F3F;
    while(right < str.size()) {
        char ch = str[right++];
        if (need.find(ch) != need.end()) {
            ++window[ch];
            if (need[ch] == window[ch]) ++valid;
        }

        while(valid == need.size()) {
            if (right - left < len) {
                len = right - left;
                start = left;
            }

            char de = str[left++];
            if (need.find(de) != need.end()) {
                if (need[de] == window[de]) --valid;
                --window[de];
            }
        }
    }
    return len == 0x3F3F3F3F ? "" : str.substr(start, len);
}
```

### [字符串排列](https://leetcode.cn/problems/permutation-in-string)

给你两个字符串 `s1` 和 `s2`，写一个函数来判断 `s2` 是否包含 `s1` 的排列。如果是，返回 `true`；否则，返回 `false`

```cpp
bool checkInclusion(string s1, string s2) {
    unordered_map<char, int> need, window;
    for (auto& ch : s1) ++need[ch];

    size_t left = 0, right = 0, valid = 0;
    while(right < s2.size()) {
        char ch = s2[right++];
        if (need.find(ch) != need.end()) {
            window[ch]++;
            if (need[ch] == window[ch]) ++valid;
        }

        while(valid == need.size()) {
            if (right - left == s1.size()) return true;

            char d = s2[left++];
            if (need.count(d)) {
                if (need[d] == window[d]) --valid;
                --window[d];
            }
        }
    }
    return false;
}
```

### [找到字符串中所有字母异位词](https://leetcode.cn/problems/find-all-anagrams-in-a-string)

给定两个字符串 `s` 和 `p`，找到 `s` 中所有 `p` 的 异位词的子串，返回这些子串的起始索引。不考虑答案输出的顺序。

异位词:指由相同字母重排列形成的字符串（包括相同的字符串）。

```cpp
vector<int> findAnagrams(string str, string pattern) {
    unordered_map<char, int> need, window;
    vector<int> res{};

    for(auto& ch : pattern) ++need[ch];

    size_t left = 0, right = 0, valid = 0;
    while(right < str.size()) {
        char ch = str[right++];
        if (need.find(ch) != need.end()) {
            ++window[ch];
            if (need[ch] == window[ch]) ++valid;
        }

        while(valid == need.size()) {
            if (right - left == pattern.size())
                res.push_back(left);

            char de = str[left++];
            if (need.find(de) != need.end()) {
                if (need[de] == window[de]) --valid;
                --window[de];
            }
        }
    }
    return res;
}
```

### [无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)

给定一个字符串`s`，请你找出其中不含有重复字符的最长子串的长度

```cpp
int lengthOfLongestSubstring(string s) {
    unordered_set<char> uset;

    int left = 0, right = 0, length = 0;
    while(right < s.size()) {
        while(uset.find(s[right]) != uset.end()) {
            uset.erase(s[left++]);
        }
        
        int new_len = right - left + 1;
        if (new_len > length) length = new_len;
        uset.insert(s[right++]);
    }

    return length;
}
```

## Graph

### 建图

```cpp
vector<vector<int>> buildGraph(int num, vector<vector<int>>& prerequisites) {
    // num - 总节点个数
    vector<vector<int>> graph = vector<vector<int>>(num, vector<int>());
    
    // Add edge
    for (auto& edge : prerequisites) {
        int from = edge[0], to = edge[1];
        graph[from].push_back(to);
    }
    return graph;
}
```

### [所有可能的路径](https://leetcode.cn/problems/all-paths-from-source-to-target/)

图的遍历

给你一个有`n`个节点的 有向无环图（DAG），请你找出所有从节点 `0` 到节点 `n-1` 的路径并输出（不要求按特定顺序）

`graph[i]` 是一个从节点 `i` 可以访问的所有节点的列表（即从节点 `i` 到节点 `graph[i][j]`存在一条有向边）

```cpp
vector<vector<int>> allPathsSourceTarget(vector<vector<int>>& graph) {
    vector<vector<int>> paths;
    vector<int> path;

    int num = graph.size(); // 该图中的总节点个数

    function<void(int)> traverse = [&](int x) {
        path.push_back(x);

        if(x == num - 1) paths.push_back(path);

        for(int v : graph[x]) traverse(v);
        path.pop_back();
    };

    traverse(0);
    return paths;
}
```

### [课程表](https://leetcode.cn/problems/course-schedule/)

图中环的检测

```cpp
bool DFS(int num, vector<vector<int>>& prerequisites) {
    vector<bool> visited(num, false);
    vector<bool> on_path(num, false);

    vector<vector<int>> graph = buildGraph(num, prerequisites);

    bool has_cycle = false;

    function<void(int)> traverse = [&](int x) {
        if(on_path[x]) {
            has_cycle = true;
            return;
        }
        if (visited[x] || has_cycle) return;

        visited[x] = true;
        on_path[x] = true;
        for (int k : graph[x]) traverse(k);
        on_path[x] = false;
    };

    for(size_t i = 0; i < graph.size(); ++i) traverse(i);
    return has_cycle;
}
```

### [课程表 II](https://leetcode.cn/problems/course-schedule-ii)

```cpp
/*
  DFS: 逆序遍历的结果，就是拓扑排序的结果
  因为一个任务完成必须等待它依赖的所有任务都完成后才能开始执行
 */
vector<int> findOrder(int num, vector<vector<int>>& prerequisites) {
    vector<vector<int>> graph = buildGraph(num, prerequisites);

    bool has_cycle = false;
    vector<bool> visited(num, false);
    vector<bool> on_path(num, false);
    vector<int> order;

    function<void(int)> traverse = [&](int x) {
        if(on_path[x]) has_cycle = true;
        if (visited[x] || has_cycle) return;

        on_path[x] = true;
        visited[x] = true;

        for(const auto& node : graph[x]) traverse(node);
        order.push_back(x);
        on_path[x] = false;
    }

    for(size_t i = 0; i < num; ++i) traverse(i);

    if(has_cycle) return {};
    reverse(order.begin(), order.end());
    vector<int> res(order.begin(), order.begin() + num);
    return res;
}

/*
 BFS: in_dgree 为 0 的节点入队，并执行 BFS 循环，不断弹出队列中的节点，减少相邻节点的入度，将入度为 0 的节点入队 
 如果最终所有节点都被遍历过（cnt == num），则说明不存在环，否则存在环
 ==图的遍历 都需要 `visited` 数组防止走回头路，这里的 BFS 算法其实是通过 `indegree` 数组实现的 `visited` 数组的作用，只有入度为 0 的节点才能入队，从而保证不会出现死循环==
*/
bool canFinish(int num, vector<vector<int>>& prerequisites) {
    function<void(vector<vector<int>>&, vector<int>&)> buildGraph = [&] (vector<vector<int>>& graph, vector<int>& in_dgree) {
        for(const auto& edge : prerequisites) {
            int from = edge[1], to = edge[0];
            graph[from].push_back(to);
            in_dgree[to]++;
        }
    };
    
    // graph, 邻接表 graph[curr] 表示邻居节点
    vector<vector<int>> graph(num, vector<int>());
    vector<int> in_dgree(num, 0);
    buildGraph(graph, in_dgree);

    // 入度为 0 的队列
    queue<int> que;
    for(size_t i = 0; i < num; ++i) if (in_dgree[i] == 0) que.push(i);
    
    // 统计遍历过的节点
    int cnt = 0;
    while(!que.empty()) {
        int curr = que.front();
        que.pop();
        ++cnt;

        for(int node : graph[curr]) {
            --in_dgree[curr];
            if(in_dgree[curr] == 0) que.push(node);
        }
    }
    return cnt == num;
}
```

**对于加权图的场景，需要使用优先队列『自动排序』特性，将路径权重较小的节点排在队列前面，以此为基础施展 BFS 算法，即 `Dijkstra` 算法**

## 回溯

- 组合问题：`N` 个数里面按一定规则找出 `k` 个数的集合
- 排列问题：`N` 个数按照一定规则全排列，有几种排列方式
- 切割问题：一个字符串按照一定规则有几种切割方式
- 子集问题：`N` 个数的集合中，有多少符合条件的子集
- 棋盘问题：`N` 皇后、数独等

**组合问题不强调元素顺序，而排列问题强调元素顺序**

### [组合](https://leetcode.cn/problems/combinations)

给定两个整数`n`和`k`，返回范围 `[1, n]`中所有可能的 `k` 个数的组合

你可以按 任何顺序 返回答案

```cpp
vector<vector<int>> combine(int n, int k) {
    vector<vector<int>> res;
    vector<int> path;

    function<void(int)> backtrace = [&] (int idx) {
        if (idx > k && path.size() >= k) {
            res.push_back(path);
            return;
        }

        for (int i = idx; i <= n; ++i) {
            path.push_back(i);
            backtrace(i + 1);
            path.pop_back();
        }
    };
    
    backtrace(1);
    return res;
}
```

### [组合总和](https://leetcode.cn/problems/combination-sum)

给你一个 无重复元素 的整数数组 `candidates` 和一个目标整数 `target` ，找出 `candidates` 中可以使数字和为目标数 `target` 的 所有 不同组合 ，并以列表形式返回

你可以按 任意顺序 返回这些组合

`candidates` 中的 同一个 数字可以 无限制重复被选取 。如果至少一个数字的被选数量不同，则两种组合是不同的。

对于给定的输入，保证和为 `target` 的不同组合数少于 150 个

```cpp
vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    vector<vector<int>> res{};
    vector<int> path{};
    int sum = 0;

    function<void(int)> backtrace = [&] (int idx) {
        if (sum > target) return;

        if (sum == target) {
            res.push_back(path);
            return;
        }

        for (int i = idx; i < candidates.size(); ++i) {
            sum += candidates[i];
            path.push_back(candidates[i]);
            backtrace(i);
            path.pop_back();
            sum -= candidates[i];
        }
    };

    backtrace(0);
    return res;
}
```

### [组合总和 II](https://leetcode.cn/problems/combination-sum-ii)

给定一个候选人编号的集合 `candidates` 和一个目标数 `target`，找出 `candidates` 中所有可以使数字和为 `target` 的组合

`candidates`中的每个数字在每个组合中只能使用 一次

```cpp
vector<vector<int>> combinationSum2(vector<int>& candidates, int target) {
    vector<vector<int>> res{};
    vector<int> path{};
    vector<bool> used(candidates.size(), false);
    int sum = 0;
    
    // 去重: sort + if(i > 0 && candidates[i] == candidates[i - 1] && used[i - 1] = false) continue;
    sort(candidates.begin(), candidates.end());
    
    function<void(int)> backtrace = [&](int idx) {
        if (sum > target) return;
        if (sum == target) {
            res.push_back(path);
            return;
        }

        for(int i = idx; i < candidates.size(); ++i) {
            if (i > 0 && candidates[i] == candidates[i - 1] && used[i - 1] == false) continue;

            used[i] = true;
            sum += candidates[i];
            path.push_back(candidates[i]);
            backtrace(i + 1);
            path.pop_back();
            sum -= candidates[i];
            used[i] = false;
        }
    };

    backtrace(0);
    return res;
}
```

### [组合总和 III](https://leetcode.cn/problems/combination-sum-iii)

找出所有相加之和为 n 的 k 个数的组合，且满足下列条件：

- 只使用数字1到9
- 每个数字 最多使用一次

返回 所有可能的有效组合的列表

该列表不能包含相同的组合两次，组合可以以任何顺序返回

```cpp
vector<vector<int>> combinationSum3(int k, int n) {
    vector<vector<int>> res{};
    vector<int> path;
    int sum = 0;

    function<void(int)> backtrace = [&] (int idx) {
        if (path.size() == k) {
            if (sum == n)
                res.push_back(path);
            return;
        }

        for (int i = idx; i <= 9; ++i) {
            sum += i;
            path.push_back(i);
            backtrace(i + 1);
            sum -= i;
            path.pop_back();
        }
    };

    backtrace(1);
    return res;
}
```

### [电话号码的字母组合](https://leetcode.cn/problems/letter-combinations-of-a-phone-number)

给定一个仅包含数字 2-9 的字符串，返回所有它能表示的字母组合

答案可以按 任意顺序 返回

`1` 不对应任何字母, 数字到字母的映射与电话按键相同

```cpp
vector<string> letterCombinations(string digits) {
    if (digits.empty()) return {};

    const string kLetterMap[10] = {
        "", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"
    };

    vector<string> res{};
    string path;

    function<void(int)> backtrace = [&] (int idx) {
        if (idx >= digits.size()) {
            res.push_back(path);
            return;
        }

        string letter = kLetterMap[digits[idx] - '0'];
        for (size_t i = 0; i < letter.size(); ++i) {
            path.push_back(letter[i]);
            backtrace(idx + 1);
            path.pop_back();
        }
    };

    backtrace(0);
    return res;
}
```

### [分割回文串](https://leetcode.cn/problems/palindrome-partitioning)

给你一个字符串 s，请你将 s 分割成一些子串，使每个子串都是 回文串

返回 s 所有可能的分割方案

> 字符串 abcdef
>
> 组合问题：选取一个 `a` 后，在 `bcdef` 中再去选取第二个，以此类推
>
> 切割问题：切割一个 `a` 后，在 `bcdef` 中再去切割第二段，以此类推

```cpp
bool isPalindrome (const string& str, int start, int end) {
    for (int i = start, j = end; i < j; ++i, --j) {
        if (str[i] != str[j]) return false;
    }
    return true;
}

vector<vector<string>> partition(string s) {
    vector<vector<string>> res{};
    vector<string> solution{};
    
    function<void(int)> backtrace = [&] (int idx) {
        if (idx >= s.size()) {
            res.push_back(solution);
            return;
        }

        for (int i = idx; i < s.size(); ++i) {
            if (isPalindrome(s, idx, i)) {
                string str = s.substr(idx, i - idx + 1);
                solution.push_back(str);
            } else continue;

            backtrace(i + 1);
            solution.pop_back();
        }
    };

    backtrace(0);
    return res;
}
```

- [复原 IP 地址](https://leetcode.cn/problems/restore-ip-addresses)

### [子集](https://leetcode.cn/problems/subsets)

给你一个整数数组`nums`，数组中的元素 互不相同

返回该数组所有可能的子集（幂集）

解集 不能 包含重复的子集。你可以按 任意顺序 返回解集

```cpp
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> res {};
    vector<int> path{};
    vector<bool> used(nums.size(), false);

    function<void(int)> backtrace = [&] (int idx) {
        res.push_back(path);

        if (idx >= nums.size()) return;

        for (int i = idx; i < nums.size(); ++i) {
            path.push_back(nums[i]);
            backtrace(i + 1);
            path.pop_back();
        }
    };

    backtrace(0);
    return res;
}
```

### [子集II](https://leetcode.cn/problems/subsets-ii)

给你一个整数数组`nums` ，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）

解集 不能 包含重复的子集

返回的解集中，子集可以按 任意顺序 排列

```cpp
vector<vector<int>> subsetsWithDup(vector<int>& nums) {
    vector<vector<int>> res{};
    vector<int> path{};
    vector<bool> used(nums.size(), false);

    sort(nums.begin(), nums.end());

    function<void(int)> backtrace = [&] (int idx) {
        res.push_back(path);

        if (idx >= nums.size()) return;
        
        for (int i = idx; i < nums.size(); ++i) {
            if (i > 0 && nums[i] == nums[i - 1] && used[i - 1] == false) continue;

            used[i] = true;
            path.push_back(nums[i]);
            backtrace(i + 1);
            path.pop_back();
            used[i] = false;
        }
    };

    backtrace(0);
    return res;
}
```

### **[递增子序列](https://leetcode.cn/problems/non-decreasing-subsequences)**

给你一个整数数组`nums`，找出并返回所有该数组中不同的递增子序列，递增子序列中 至少有两个元素

你可以按 任意顺序 返回答案

数组中可能含有重复元素，如出现两个整数相等，也可以视作递增序列的一种特殊情况

```cpp
vector<vector<int>> findSubsequences(vector<int>& nums) {
    vector<vector<int>> res{};
    vector<int> path{};

    function<void(int)> backtrace = [&] (int idx) {
        if (path.size() >= 2) res.push_back(path);

        unordered_set<int> uset; // 使用 set 对本层元素进行去重
        for (int i = idx; i < nums.size(); ++i) {
            if ((!path.empty() && nums[i] < path.back())
                || uset.find(nums[i]) != uset.end()) {
                continue;
            }
            
            uset.insert(nums[i]);
            path.push_back(nums[i]);
            backtrace(i + 1);
            path.pop_back();
        }
    };

    backtrace(0);
    return res;
}
```

### [全排列](https://leetcode.cn/problems/permutations)

给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列

你可以 按任意顺序 返回答案

```cpp
vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> res;
    vector<int> path;
    unordered_set<int> uset; // 去重

    function<void()> backtrace = [&] () {
        if (path.size() == nums.size()) {
            res.push_back(path);
            return;
        }

        for (int i = 0; i < nums.size(); ++i) {
            if (uset.find(nums[i]) != uset.end()) continue;

            uset.insert(nums[i]);
            path.push_back(nums[i]);
            backtrace();
            path.pop_back();
            uset.erase(nums[i]);
        }
    };

    backtrace();
    return res;
}
```

### [全排列 II](https://leetcode.cn/problems/permutations-ii)

给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列

```cpp
vector<vector<int>> permuteUnique(vector<int>& nums) {
    vector<vector<int>> res;
    vector<int> path;
    vector<bool> used(nums.size(), false);

    sort(nums.begin(), nums.end());  // 排序

    function<void()> backtrace = [&]() {
        if (path.size() == nums.size()) {
            res.push_back(path);
            return ;
        }

        for (int i = 0; i < nums.size(); ++i) {
            if (i > 0 && nums[i - 1] == nums[i] && used[i - 1] == false) continue; // 去重

            if (used[i] == false) {  // 必须在当前位没有使用过的情况下开启递归
                used[i] = true;
                path.push_back(nums[i]);
                backtrace();
                path.pop_back();
                used[i] = false;
            }
        }
    };

    backtrace();
    return res;
}
```

## 动态规划

**==动态规划问题的一般形式就是求最值==, 重叠子问题、最优子结构、状态转移方程就是动态规划三要素**

**递归算法的时间复杂度怎么计算？就是用子问题个数乘以解决一个子问题需要的时间**

**动态规划的通用技巧：数学归纳思想**

### [斐波那契数列](https://leetcode.cn/problems/fibonacci-number/)

该数列由`0`和`1`开始，后面的每一项数字都是前面两项数字的和

```cpp
int fib(int n) {
    if (n == 0) return 0;

    vector<int> dp(n + 1);
    dp[0] = 0;
    dp[1] = 1;

    for (int i = 2; i <= n; ++i)
        dp[i] = dp[i - 1] + dp[i - 2];
    return dp[n];
}
```

### [零钱兑换](https://leetcode.cn/problems/coin-change/)

给你一个整数数组`coins`，表示不同面额的硬币；以及一个整数`amount`，表示总金额

计算并返回可以凑成总金额所需的 最少的硬币个数, 如果没有任何一种硬币组合能组成总金额，返回 -1

你可以认为每种硬币的数量是无限的

```cpp
int coinChange(vector<int>& coins, int amount) {
    // dp[i] 表示装满背包容量为 i 的背包，所需的最少硬币个数
    // dp[i] = min(dp[i - coins[i]] + 1, dp[i])
    const int kDEFAULT = 0x3F3F3F3F;
    vector<int> dp(amount + 1, kDEFAULT);
    dp[0] = 0;

    for (int i = 1; i <= amount; ++i) {
        for (auto coin : coins) {
            if (coin <= i) {
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }
    } 
    return dp[amount] == kDEFAULT ? -1 : dp[amount];
}
```

### [最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/)

给你一个整数数组 nums ，找到其中最长严格递增子序列的长度

子序列 是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。例如，[3,6,2,7] 是数组 [0,3,1,6,2,2,7] 的子序列

```cpp
int lengthOfLIS(vector<int>& nums) {
    if (nums.empty()) return 0;

    // d[i] 表示以 nums[i - 1] 为结尾的最长子序列的长度
    vector<int> dp(nums.size(), 1); // dp[i] 置 1, 因为每一个元素都可以单独成为子序列

    // 
    for (int i = 1; i < nums.size(); ++i) {
        for (int j = 0; j < i; ++j) {
            if (nums[j] < nums[i]) // 严格递增
                dp[i] = max(dp[i], dp[j] + 1);
        }
    }

    return *max_element(dp.begin(), dp.end());
}
```

### [下降路径最小和](https://leetcode.cn/problems/minimum-falling-path-sum/)

给你一个 `n x n` 的方形整数数组`matrix`，请你找出并返回通过`matrix`的下降路径的最小和

下降路径 可以从第一行中的任何元素开始，并从每一行中选择一个元素。在下一行选择的元素和当前行所选元素最多相隔一列（即位于正下方或者沿对角线向左或者向右的第一个元素）

```cpp
int minFallingPathSum(vector<vector<int>>& matrix) {
    if (matrix.empty()) return 0;

    // dp[i][j] 表示以 nums[i][j] 最后一个点的最小路径和
    // dp[i][j] = matrix[i][j] + 
    //              min(dp[i - 1][j - 1], dp[i - 1][j], dp[i - 1][j + 1]);
    vector<vector<int>> dp(matrix.size(), vector<int>(matrix[0].size()));
    
    function<int(int, int)> get = [&](int i, int j) {
        if (j < 0 || j >= matrix[i].size()) return 0x3F3F3F3F;
        return dp[i][j];
    };

    // 初始化
    for (int j = 0; j < matrix[0].size(); ++j) dp[0][j] = matrix[0][j];
    
    for (int i = 1; i < matrix.size(); ++i) {
        for (int j = 0; j < matrix[0].size(); ++j) {
            dp[i][j] = matrix[i][j] + min(min(get(i - 1, j - 1), get(i - 1, j)), get(i - 1, j + 1));
        }
    }

    int rc = 0x3F3F3F3F;
    for (int j = 0; j < matrix[0].size(); ++j) {
        rc = min(rc, dp[matrix.size() - 1][j]);
    }
    return rc;
}
```

### [单词拆分](https://leetcode.cn/problems/word-break/description/)

给你一个字符串 `s` 和一个字符串列表 `wordDict` 作为字典。请你判断是否可以利用字典中出现的单词拼接出`s`

不要求字典中出现的单词全部都使用，并且字典中的单词可以重复使用

```cpp
bool wordBreak(string s, vector<string>& wordDict) {
    // unordered_set<string> uset;
    // for (const auto& word : wordDict) uset.insert(word);
    unordered_set<string> uset(wordDict.begin(), wordDict.end()); // 代替 vector 查找，加快查询速度

    // dp[i] 表示以 i - 1 为结尾的子串 s[0, i) 是否可以满足条件
    vector<bool> dp(s.size() + 1, false);
    dp[0] = true;

    for (int i = 1; i <= s.size(); ++i) { // 遍历背包
        for (int j = 0; j < i; ++j) { // 遍历物品
            if (dp[j] && 
                    uset.find(s.substr(j, i - j)) != uset.end()) 
                dp[i] = true;
        }
    }

    return dp[s.size()];
}
```

### [两个字符串的删除操作](https://leetcode.cn/problems/delete-operation-for-two-strings/)

给定两个单词 word1 和 word2 ，返回使得 word1 和  word2 相同所需的最小步数

每步 可以删除任意一个字符串中的一个字符

```cpp
int minDistance(string word1, string word2) {
    if (word1.empty()) return word2.size();
    if (word2.empty()) return word1.size();

    // dp[i][j] : 使得 word1[0, i - 1) 与 word2[0, j - 1) 相同的最小步数
    vector<vector<int>> dp(word1.size() + 1, vector<int>(word2.size() + 1));
    dp[0][0] = 0;
    for (int i = 0; i <= word1.size(); ++i) dp[i][0] = i;
    for (int j = 0; j <= word2.size(); ++j) dp[0][j] = j;

    for (int i = 1; i <= word1.size(); ++i) {
        for (int j = 1; j <= word2.size(); ++j) {
            if (word1[i - 1] == word2[j - 1]) // dp[i][j]: word1[i - 1] vs word2[j - 1]
                dp[i][j] = dp[i - 1][j - 1];
            else
                dp[i][j] = min(dp[i][j - 1], dp[i - 1][j]) + 1;
        }
    }

    return dp[word1.size()][word2.size()];
}
```

- 删除之后，得到的结果就是最长公共子序列

### [编辑距离](https://leetcode.cn/problems/edit-distance/)

给你两个单词 word1 和 word2， 请返回将 word1 转换成 word2 所使用的最少操作数

可以对一个单词进行如下三种操作：

- 插入一个字符
- 删除一个字符
- 替换一个字符

```cpp
int minDistance(string word1, string word2) {
    // dp[i][j]: 将 word1[0, i - 1) 转换成 word2[0, j - 1) 所需最小操作数
    vector<vector<int>> dp(word1.size() + 1, vector<int>(word2.size() + 1));
    dp[0][0] = 0;

    for (int i = 0; i <= word1.size(); ++i) dp[i][0] = i;
    for (int j = 0; j <= word2.size(); ++j) dp[0][j] = j;

    for (int i = 1; i <= word1.size(); ++i) {
        for (int j = 1; j <= word2.size(); ++j) {
            if (word1[i - 1] == word2[j - 1])
                dp[i][j] = dp[i - 1][j - 1];
            else
                dp[i][j] = min(
                    min(dp[i - 1][j] /* 删除 */, dp[i][j - 1]), /* 插入*/
                    dp[i - 1][j - 1] /* 修改 */) + 1;
        }
    }

    return dp[word1.size()][word2.size()];
}
```

### [最长公共子序列](https://leetcode.cn/problems/longest-common-subsequence/)

给定两个字符串 text1 和 text2，返回这两个字符串的最长 公共子序列 的长度

如果不存在 公共子序列 ，返回 `0`

一个字符串的 子序列 是指这样一个新的字符串：它是由原字符串在不改变字符的相对顺序的情况下删除某些字符（也可以不删除任何字符）后组成的新字符串

两个字符串的 公共子序列 是这两个字符串所共同拥有的子序列

```cpp
int longestCommonSubsequence(string s1, string s2) {
    if (s1.empty() || s2.empty()) return 0;

    // dp[i]][j]: 子串 s1[0, i - 1) 与子串 s2[0, j - 1) 的最长公共子序列长度
    vector<vector<int>> dp(s1.size() + 1, vector<int>(s2.size() + 1));
    dp[0][0] = 0;

    for (int i = 0; i <= s1.size(); ++i) dp[i][0] = 0;
    for (int j = 0; j <= s2.size(); ++j) dp[0][j] = 0;

    for (int i = 1; i <= s1.size(); ++i) {
        for (int j = 1; j <= s2.size(); ++j) {
            if (s1[i - 1] == s2[j - 1])
                dp[i][j] = dp[i - 1][j - 1] + 1;
            else
                dp[i][j] = max(dp[i][j - 1], dp[i - 1][j]);
        }
    }

    return dp[s1.size()][s2.size()];
}
```

### [最大子数组和](https://leetcode.cn/problems/maximum-subarray/)

给你一个整数数组`nums`，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和

子数组是数组中的一个连续部分

```cpp
int maxSubArray(vector<int>& nums) {
    // dp[i] 表示以 nums[i - 1] 为结尾且具有最大和的子数组的最大和（即满足条件的子数组的最大和）
    vector<int> dp(nums.size() + 1);
    dp[0] = -0x3F3F3F3F;

    for (size_t i = 1; i <= nums.size(); ++i) {
        dp[i] = max(nums[i - 1], dp[i - 1] + nums[i - 1]);
    }

    return *max_element(dp.begin(), dp.end());
}
```

## 贪心 Greedy

[无重叠区间](https://leetcode.cn/problems/non-overlapping-intervals/)

给定一个区间的集合`intervals`，其中 `intervals[i] = [starti, endi]`

返回 需要移除区间的最小数量，使剩余区间互不重叠

```cpp
/* Dynamic Planning: Timeout
 * 本质：最长上升子序列，即找到最多的无重叠区间
 */
int eraseOverlapIntervals(vector<vector<int>>& intervals) {
    if (intervals.empty()) return 0;

    sort(intervals.begin(), intervals.end(), 
        [](vector<int>& u, vector<int>& v){
            return u[0] < v[0];
    });

        
    // f[i] 表示以 i 区间为最后一个区间时, 可选出的最大区间数量
    vector<int> f(intervals.size(), 1);

    for (int i = 1; i < intervals.size(); ++i) {
        for (int j = 0; j < i; ++j) {
            if (intervals[i][0] >= intervals[j][1])
                f[i] = max(f[i], f[j] + 1);
        }
    }
    return intervals.size() - *max_element(f.begin(), f.end());
}

/* Greedy */
int eraseOverlapIntervals(vector<vector<int>>& intervals) {
    if (intervals.empty()) return 0;

    sort(intervals.begin(), intervals.end(), 
        [](vector<int>& u, vector<int>& v){
            return u[1] < v[1];
    });
    
    int cnt = 1; // 至少有一个区间
    int curr_end = intervals[0][1]; // The end of Current Interval
    for (const auto& interval : intervals) {
        int next_start = interval[0]; // The start of Next Interval

        if (next_start >= curr_end) { // Find an valid interval
            curr_end = interval[1];
            ++cnt;
        }
    }

    return intervals.size() - cnt;
}
```

### [用最少数量的箭矢引爆气球](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)

一支弓箭可以沿着`x`轴从不同点 完全垂直 地射出

在坐标`x`处射出一支箭，若有一个气球的直径的开始和结束坐标为`xstart`，`xend`， 且满足`xstart ≤ x ≤ xend`，则该气球会被引爆

可以射出的弓箭的数量 没有限制 。 弓箭一旦被射出之后，可以无限地前进

给你一个数组 `points` ，返回引爆所有气球所必须射出的 最小 弓箭数

```cpp
int findMinArrowShots(vector<vector<int>>& points) {
    if (points.empty()) return 0;

    // 按照末尾位置从小到大排序
    sort(points.begin(), points.end(),
            [] (vector<int>& u, vector<int>& v) {
                return u[1] < v[1];
            });
    
    int cnt = 1;  // 最少需要一支箭
    int curr_end = points[0][1], next_start;  // 
    for (const auto& point : points) {
        next_start = point[0];

        if (curr_end < next_start) { // 只有两个气球不挨着时，才需要另一支箭，并更新 curr_end
            ++cnt;
            curr_end = point[1];
        }
    }

    return cnt;
}
```

### [跳跃游戏](https://leetcode.cn/problems/jump-game/)

给你一个非负整数数组 `nums`，你最初位于数组的 第一个下标

数组中的每个元素代表你在该位置可以跳跃的最大长度

判断你是否能够到达最后一个下标，如果可以，返回 `true` ；否则，返回 `false`

```cpp
bool canJump(vector<int>& nums) {
    if (nums.empty()) return true;

    // 所能达到最远的位置
    int cover = nums[0];

    for (int i = 0; i <= cover; ++i) {
        if (cover >= nums.size() - 1) return true;
        if (cover <= nums[i] + i) cover = nums[i] + i;
    }
    return false;
}
```

### [视频拼接](https://leetcode.cn/problems/video-stitching/)

使用数组 `clips` 描述所有的视频片段，其中 `clips[i] = [starti, endi]` 表示：某个视频片段开始于 `starti` 并于 `endi` 结束

需要将这些片段进行再剪辑，并将剪辑后的内容拼接成覆盖整个运动过程的片段（`[0, time]`）。

返回所需片段的最小数目，如果无法完成该任务，则返回`-1`

```cpp
int videoStitching(vector<vector<int>>& clips, int time) {
    if (time <= 0) return -1;

    // 按起点从小到大排序（起点相同时，按右端点降序排序）
    sort (clips.begin(), clips.end(),
        [] (const vector<int>& u, const vector<int>& v) {
            return u[0] == v[0] ? u[1] < v[1] : u[0] < v[0];
        });

    int i = 0, cnt = 0, curr_end = 0, next_end = -0x3F3F3F3F;
    while (i < clips.size() && clips[i][0] <= curr_end) {
        // 当多个“下一区间的左端点” 位于 “当前区间右端点“ 内部时，找到最大的右端点
        while(i < clips.size() && curr_end >= clips[i][0]) {
            next_end = max(next_end, clips[i][1]);
            ++i;
        }

        curr_end = next_end;
        ++cnt;

        if (curr_end >= time) return cnt; //  类似于跳跃游戏，以满足要求，直接返回
    }
    return -1;
}
```
