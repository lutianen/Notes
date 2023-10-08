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

## List

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

## 单调栈

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

```cpp

```
