# DSA Java Handbook

This handbook contains the exact 50 problems you listed, along with:
- A short **approach**
- A clean **Java solution**
- Typical **time/space complexity**

---

## Problem List (As Requested)

### 1) Arrays (Most Asked)
1. Two Sum  
2. Best Time to Buy and Sell Stock  
3. Maximum Subarray (Kadane’s Algorithm)  
4. Majority Element  
5. Merge Intervals  
6. Product of Array Except Self  
7. Longest Consecutive Sequence  
8. Subarray Sum Equals K  

### 2) Sliding Window
9. Longest Substring Without Repeating Characters  
10. Longest Repeating Character Replacement  
11. Minimum Window Substring  
12. Maximum Sum Subarray of Size K  
13. Count Subarrays with Sum K  

### 3) Two Pointer Problems
14. Container With Most Water  
15. Trapping Rain Water  
16. 3Sum  
17. Remove Duplicates from Sorted Array  
18. Move Zeroes  

### 4) Binary Search (Very Important)
19. Binary Search  
20. Search in Rotated Sorted Array  
21. Find Peak Element  
22. First and Last Position of Element  
23. Koko Eating Bananas  
24. Median of Two Sorted Arrays  

### 5) Stack / Monotonic Stack
25. Valid Parentheses  
26. Next Greater Element  
27. Daily Temperatures  
28. Largest Rectangle in Histogram  
29. Min Stack  

### 6) Linked List
30. Reverse Linked List  
31. Detect Cycle in Linked List  
32. Merge Two Sorted Lists  
33. Remove Nth Node from End  
34. LRU Cache  

### 7) Trees (Very Important)
35. Binary Tree Inorder Traversal  
36. Level Order Traversal  
37. Maximum Depth of Binary Tree  
38. Diameter of Binary Tree  
39. Lowest Common Ancestor  
40. Validate Binary Search Tree  

### 8) Graph Problems
41. Number of Islands  
42. Clone Graph  
43. Course Schedule  
44. BFS / DFS Traversal  

### 9) Heap / Priority Queue
45. Kth Largest Element in Array  
46. Top K Frequent Elements  
47. Merge K Sorted Lists  

### 10) Dynamic Programming (Medium Level)
48. Climbing Stairs  
49. Coin Change  
50. Longest Increasing Subsequence  

---

## 1) Arrays

### 1. Two Sum
**Approach:** Use `HashMap<value, index>`. For each number, check if `target - num` already exists.  
**Complexity:** O(n) time, O(n) space.

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (map.containsKey(need)) return new int[]{map.get(need), i};
        map.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
```

### 2. Best Time to Buy and Sell Stock
**Approach:** Track minimum price so far and maximum profit at each day.  
**Complexity:** O(n) time, O(1) space.

```java
public int maxProfit(int[] prices) {
    int min = Integer.MAX_VALUE, ans = 0;
    for (int p : prices) {
        min = Math.min(min, p);
        ans = Math.max(ans, p - min);
    }
    return ans;
}
```

### 3. Maximum Subarray (Kadane’s)
**Approach:** At each index, either start new subarray or extend previous one.  
**Complexity:** O(n) time, O(1) space.

```java
public int maxSubArray(int[] nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);
        best = Math.max(best, cur);
    }
    return best;
}
```

### 4. Majority Element
**Approach:** Boyer-Moore Voting keeps candidate and count.  
**Complexity:** O(n) time, O(1) space.

```java
public int majorityElement(int[] nums) {
    int candidate = 0, count = 0;
    for (int n : nums) {
        if (count == 0) candidate = n;
        count += (n == candidate) ? 1 : -1;
    }
    return candidate;
}
```

### 5. Merge Intervals
**Approach:** Sort by start, merge overlapping intervals greedily.  
**Complexity:** O(n log n) time, O(n) space.

```java
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    List<int[]> res = new ArrayList<>();
    for (int[] in : intervals) {
        if (res.isEmpty() || res.get(res.size() - 1)[1] < in[0]) {
            res.add(new int[]{in[0], in[1]});
        } else {
            res.get(res.size() - 1)[1] = Math.max(res.get(res.size() - 1)[1], in[1]);
        }
    }
    return res.toArray(new int[0][]);
}
```

### 6. Product of Array Except Self
**Approach:** Build prefix product into result, then multiply suffix product in reverse pass.  
**Complexity:** O(n) time, O(1) extra space (excluding output).

```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    res[0] = 1;
    for (int i = 1; i < n; i++) res[i] = res[i - 1] * nums[i - 1];
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= suffix;
        suffix *= nums[i];
    }
    return res;
}
```

### 7. Longest Consecutive Sequence
**Approach:** Put all numbers in a set. Start counting only when `num - 1` not present.  
**Complexity:** O(n) average time, O(n) space.

```java
public int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int n : nums) set.add(n);
    int best = 0;
    for (int n : set) {
        if (!set.contains(n - 1)) {
            int cur = n, len = 1;
            while (set.contains(cur + 1)) {
                cur++;
                len++;
            }
            best = Math.max(best, len);
        }
    }
    return best;
}
```

### 8. Subarray Sum Equals K
**Approach:** Prefix sum + hashmap of frequencies. If `sum - k` seen before, add count.  
**Complexity:** O(n) time, O(n) space.

```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1);
    int sum = 0, ans = 0;
    for (int n : nums) {
        sum += n;
        ans += freq.getOrDefault(sum - k, 0);
        freq.put(sum, freq.getOrDefault(sum, 0) + 1);
    }
    return ans;
}
```

---

## 2) Sliding Window

### 9. Longest Substring Without Repeating Characters
**Approach:** Expand right pointer; shrink left while duplicate exists.  
**Complexity:** O(n) time, O(charset) space.

```java
public int lengthOfLongestSubstring(String s) {
    int[] last = new int[128];
    Arrays.fill(last, -1);
    int left = 0, ans = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (last[c] >= left) left = last[c] + 1;
        last[c] = right;
        ans = Math.max(ans, right - left + 1);
    }
    return ans;
}
```

### 10. Longest Repeating Character Replacement
**Approach:** Keep max frequency in window. If replacements needed `> k`, shrink from left.  
**Complexity:** O(n) time, O(1) space (26 chars).

```java
public int characterReplacement(String s, int k) {
    int[] count = new int[26];
    int left = 0, maxFreq = 0, ans = 0;
    for (int right = 0; right < s.length(); right++) {
        maxFreq = Math.max(maxFreq, ++count[s.charAt(right) - 'A']);
        while ((right - left + 1) - maxFreq > k) {
            count[s.charAt(left) - 'A']--;
            left++;
        }
        ans = Math.max(ans, right - left + 1);
    }
    return ans;
}
```

### 11. Minimum Window Substring
**Approach:** Expand to satisfy required counts, then shrink to find minimal valid window.  
**Complexity:** O(m + n) time, O(1) space (ASCII).

```java
public String minWindow(String s, String t) {
    if (t.length() > s.length()) return "";
    int[] need = new int[128];
    for (char c : t.toCharArray()) need[c]++;
    int required = t.length(), left = 0, start = 0, minLen = Integer.MAX_VALUE;
    for (int right = 0; right < s.length(); right++) {
        if (need[s.charAt(right)]-- > 0) required--;
        while (required == 0) {
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                start = left;
            }
            if (++need[s.charAt(left++)] > 0) required++;
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(start, start + minLen);
}
```

### 12. Maximum Sum Subarray of Size K
**Approach:** Fixed-size sliding window, add right and remove left.  
**Complexity:** O(n) time, O(1) space.

```java
public int maxSumSubarrayK(int[] nums, int k) {
    int sum = 0, ans = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        sum += nums[i];
        if (i >= k) sum -= nums[i - k];
        if (i >= k - 1) ans = Math.max(ans, sum);
    }
    return ans;
}
```

### 13. Count Subarrays with Sum K
**Approach:** Same as #8 (prefix sum frequency map).  
**Complexity:** O(n) time, O(n) space.

```java
public int countSubarraysWithSumK(int[] nums, int k) {
    Map<Integer, Integer> map = new HashMap<>();
    map.put(0, 1);
    int sum = 0, count = 0;
    for (int n : nums) {
        sum += n;
        count += map.getOrDefault(sum - k, 0);
        map.put(sum, map.getOrDefault(sum, 0) + 1);
    }
    return count;
}
```

---

## 3) Two Pointers

### 14. Container With Most Water
**Approach:** Start at both ends. Move shorter side inward to seek taller boundary.  
**Complexity:** O(n) time, O(1) space.

```java
public int maxArea(int[] height) {
    int l = 0, r = height.length - 1, ans = 0;
    while (l < r) {
        int area = Math.min(height[l], height[r]) * (r - l);
        ans = Math.max(ans, area);
        if (height[l] < height[r]) l++;
        else r--;
    }
    return ans;
}
```

### 15. Trapping Rain Water
**Approach:** Two pointers with leftMax and rightMax. Move smaller side and add trapped water.  
**Complexity:** O(n) time, O(1) space.

```java
public int trap(int[] height) {
    int l = 0, r = height.length - 1;
    int leftMax = 0, rightMax = 0, water = 0;
    while (l < r) {
        if (height[l] < height[r]) {
            leftMax = Math.max(leftMax, height[l]);
            water += leftMax - height[l];
            l++;
        } else {
            rightMax = Math.max(rightMax, height[r]);
            water += rightMax - height[r];
            r--;
        }
    }
    return water;
}
```

### 16. 3Sum
**Approach:** Sort. Fix one index and solve two-sum with two pointers, skipping duplicates.  
**Complexity:** O(n^2) time, O(1) extra space.

```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;
        int l = i + 1, r = nums.length - 1;
        while (l < r) {
            int sum = nums[i] + nums[l] + nums[r];
            if (sum == 0) {
                res.add(Arrays.asList(nums[i], nums[l], nums[r]));
                while (l < r && nums[l] == nums[l + 1]) l++;
                while (l < r && nums[r] == nums[r - 1]) r--;
                l++;
                r--;
            } else if (sum < 0) l++;
            else r--;
        }
    }
    return res;
}
```

### 17. Remove Duplicates from Sorted Array
**Approach:** Slow pointer tracks write position for unique elements.  
**Complexity:** O(n) time, O(1) space.

```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int write = 1;
    for (int read = 1; read < nums.length; read++) {
        if (nums[read] != nums[read - 1]) nums[write++] = nums[read];
    }
    return write;
}
```

### 18. Move Zeroes
**Approach:** Place non-zero elements in front, then fill rest with zeroes.  
**Complexity:** O(n) time, O(1) space.

```java
public void moveZeroes(int[] nums) {
    int write = 0;
    for (int n : nums) if (n != 0) nums[write++] = n;
    while (write < nums.length) nums[write++] = 0;
}
```

---

## 4) Binary Search

### 19. Binary Search
**Approach:** Standard binary search on sorted array.  
**Complexity:** O(log n) time, O(1) space.

```java
public int search(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return m;
        if (nums[m] < target) l = m + 1;
        else r = m - 1;
    }
    return -1;
}
```

### 20. Search in Rotated Sorted Array
**Approach:** One half is always sorted. Decide which half to search using boundaries.  
**Complexity:** O(log n) time, O(1) space.

```java
public int searchRotated(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return m;
        if (nums[l] <= nums[m]) {
            if (nums[l] <= target && target < nums[m]) r = m - 1;
            else l = m + 1;
        } else {
            if (nums[m] < target && target <= nums[r]) l = m + 1;
            else r = m - 1;
        }
    }
    return -1;
}
```

### 21. Find Peak Element
**Approach:** If middle is smaller than right neighbor, peak lies right; else left including mid.  
**Complexity:** O(log n) time, O(1) space.

```java
public int findPeakElement(int[] nums) {
    int l = 0, r = nums.length - 1;
    while (l < r) {
        int m = l + (r - l) / 2;
        if (nums[m] < nums[m + 1]) l = m + 1;
        else r = m;
    }
    return l;
}
```

### 22. First and Last Position of Element
**Approach:** Binary search twice: left boundary and right boundary.  
**Complexity:** O(log n) time, O(1) space.

```java
public int[] searchRange(int[] nums, int target) {
    return new int[]{bound(nums, target, true), bound(nums, target, false)};
}
private int bound(int[] nums, int target, boolean first) {
    int l = 0, r = nums.length - 1, ans = -1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) {
            ans = m;
            if (first) r = m - 1;
            else l = m + 1;
        } else if (nums[m] < target) l = m + 1;
        else r = m - 1;
    }
    return ans;
}
```

### 23. Koko Eating Bananas
**Approach:** Binary search eating speed `k` from 1..maxPile. Check if total hours <= h.  
**Complexity:** O(n log maxPile) time, O(1) space.

```java
public int minEatingSpeed(int[] piles, int h) {
    int l = 1, r = 0;
    for (int p : piles) r = Math.max(r, p);
    while (l < r) {
        int m = l + (r - l) / 2;
        long hours = 0;
        for (int p : piles) hours += (p + m - 1) / m;
        if (hours <= h) r = m;
        else l = m + 1;
    }
    return l;
}
```

### 24. Median of Two Sorted Arrays
**Approach:** Binary search partition on smaller array, ensure left parts <= right parts.  
**Complexity:** O(log(min(m,n))) time, O(1) space.

```java
public double findMedianSortedArrays(int[] a, int[] b) {
    if (a.length > b.length) return findMedianSortedArrays(b, a);
    int m = a.length, n = b.length;
    int l = 0, r = m;
    while (l <= r) {
        int i = (l + r) / 2;
        int j = (m + n + 1) / 2 - i;
        int aL = (i == 0) ? Integer.MIN_VALUE : a[i - 1];
        int aR = (i == m) ? Integer.MAX_VALUE : a[i];
        int bL = (j == 0) ? Integer.MIN_VALUE : b[j - 1];
        int bR = (j == n) ? Integer.MAX_VALUE : b[j];
        if (aL <= bR && bL <= aR) {
            if (((m + n) & 1) == 0) {
                return (Math.max(aL, bL) + Math.min(aR, bR)) / 2.0;
            } else {
                return Math.max(aL, bL);
            }
        } else if (aL > bR) r = i - 1;
        else l = i + 1;
    }
    return 0.0;
}
```

---

## 5) Stack / Monotonic Stack

### 25. Valid Parentheses
**Approach:** Push opening brackets, match and pop on closing.  
**Complexity:** O(n) time, O(n) space.

```java
public boolean isValid(String s) {
    Deque<Character> st = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') st.push(c);
        else {
            if (st.isEmpty()) return false;
            char t = st.pop();
            if ((c == ')' && t != '(') || (c == ']' && t != '[') || (c == '}' && t != '{')) return false;
        }
    }
    return st.isEmpty();
}
```

### 26. Next Greater Element
**Approach:** Traverse from right, maintain decreasing stack. Top is next greater.  
**Complexity:** O(n) time, O(n) space.

```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    Deque<Integer> st = new ArrayDeque<>();
    for (int i = n - 1; i >= 0; i--) {
        while (!st.isEmpty() && st.peek() <= nums[i]) st.pop();
        res[i] = st.isEmpty() ? -1 : st.peek();
        st.push(nums[i]);
    }
    return res;
}
```

### 27. Daily Temperatures
**Approach:** Monotonic decreasing stack of indices. On warmer day, resolve previous indices.  
**Complexity:** O(n) time, O(n) space.

```java
public int[] dailyTemperatures(int[] temp) {
    int n = temp.length;
    int[] ans = new int[n];
    Deque<Integer> st = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        while (!st.isEmpty() && temp[i] > temp[st.peek()]) {
            int idx = st.pop();
            ans[idx] = i - idx;
        }
        st.push(i);
    }
    return ans;
}
```

### 28. Largest Rectangle in Histogram
**Approach:** Monotonic increasing stack. When lower height comes, compute areas with popped bars.  
**Complexity:** O(n) time, O(n) space.

```java
public int largestRectangleArea(int[] heights) {
    int n = heights.length, ans = 0;
    Deque<Integer> st = new ArrayDeque<>();
    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];
        while (!st.isEmpty() && heights[st.peek()] > h) {
            int height = heights[st.pop()];
            int right = i;
            int left = st.isEmpty() ? -1 : st.peek();
            ans = Math.max(ans, height * (right - left - 1));
        }
        st.push(i);
    }
    return ans;
}
```

### 29. Min Stack
**Approach:** Keep main stack and min stack where min stack stores current minimum per push.  
**Complexity:** O(1) per operation.

```java
class MinStack {
    private Deque<Integer> st = new ArrayDeque<>();
    private Deque<Integer> mn = new ArrayDeque<>();
    public void push(int val) {
        st.push(val);
        mn.push(mn.isEmpty() ? val : Math.min(val, mn.peek()));
    }
    public void pop() {
        st.pop();
        mn.pop();
    }
    public int top() { return st.peek(); }
    public int getMin() { return mn.peek(); }
}
```

---

## 6) Linked List

### 30. Reverse Linked List
**Approach:** Iteratively reverse `next` pointers using `prev`, `cur`, `next`.  
**Complexity:** O(n) time, O(1) space.

```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null, cur = head;
    while (cur != null) {
        ListNode nxt = cur.next;
        cur.next = prev;
        prev = cur;
        cur = nxt;
    }
    return prev;
}
```

### 31. Detect Cycle in Linked List
**Approach:** Floyd's slow/fast pointers. If they meet, cycle exists.  
**Complexity:** O(n) time, O(1) space.

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### 32. Merge Two Sorted Lists
**Approach:** Dummy head + iterative merge choosing smaller node each step.  
**Complexity:** O(m+n) time, O(1) space.

```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), cur = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { cur.next = l1; l1 = l1.next; }
        else { cur.next = l2; l2 = l2.next; }
        cur = cur.next;
    }
    cur.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

### 33. Remove Nth Node from End
**Approach:** Two pointers with gap `n`, then remove target via previous pointer.  
**Complexity:** O(n) time, O(1) space.

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode first = dummy, second = dummy;
    for (int i = 0; i <= n; i++) first = first.next;
    while (first != null) {
        first = first.next;
        second = second.next;
    }
    second.next = second.next.next;
    return dummy.next;
}
```

### 34. LRU Cache
**Approach:** HashMap for O(1) lookup + doubly linked list for recency order.  
**Complexity:** O(1) get/put.

```java
class LRUCache {
    private final int cap;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0), tail = new Node(0, 0);

    public LRUCache(int capacity) {
        this.cap = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        Node n = map.get(key);
        if (n == null) return -1;
        remove(n);
        addFirst(n);
        return n.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) remove(map.get(key));
        if (map.size() == cap) remove(tail.prev);
        addFirst(new Node(key, value));
    }

    private void addFirst(Node n) {
        map.put(n.key, n);
        n.next = head.next;
        n.prev = head;
        head.next.prev = n;
        head.next = n;
    }

    private void remove(Node n) {
        map.remove(n.key);
        n.prev.next = n.next;
        n.next.prev = n.prev;
    }

    static class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }
}
```

---

## 7) Trees

### 35. Binary Tree Inorder Traversal
**Approach:** Iterative stack: go left, visit, then right.  
**Complexity:** O(n) time, O(h) space.

```java
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    Deque<TreeNode> st = new ArrayDeque<>();
    TreeNode cur = root;
    while (cur != null || !st.isEmpty()) {
        while (cur != null) {
            st.push(cur);
            cur = cur.left;
        }
        cur = st.pop();
        res.add(cur.val);
        cur = cur.right;
    }
    return res;
}
```

### 36. Level Order Traversal
**Approach:** BFS using queue level by level.  
**Complexity:** O(n) time, O(w) space.

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> ans = new ArrayList<>();
    if (root == null) return ans;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            level.add(node.val);
            if (node.left != null) q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
        ans.add(level);
    }
    return ans;
}
```

### 37. Maximum Depth of Binary Tree
**Approach:** Recursively compute `1 + max(leftDepth, rightDepth)`.  
**Complexity:** O(n) time, O(h) space.

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

### 38. Diameter of Binary Tree
**Approach:** For each node, diameter through node is leftHeight + rightHeight. Track global max.  
**Complexity:** O(n) time, O(h) space.

```java
private int diameter = 0;
public int diameterOfBinaryTree(TreeNode root) {
    height(root);
    return diameter;
}
private int height(TreeNode node) {
    if (node == null) return 0;
    int l = height(node.left), r = height(node.right);
    diameter = Math.max(diameter, l + r);
    return 1 + Math.max(l, r);
}
```

### 39. Lowest Common Ancestor
**Approach:** Recurse down. If both left and right return non-null, current is LCA.  
**Complexity:** O(n) time, O(h) space.

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

### 40. Validate Binary Search Tree
**Approach:** DFS with valid value range `(min, max)` for each node.  
**Complexity:** O(n) time, O(h) space.

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) && validate(node.right, node.val, max);
}
```

---

## 8) Graph Problems

### 41. Number of Islands
**Approach:** DFS/BFS from every unvisited land cell; mark visited cells.  
**Complexity:** O(m*n) time, O(m*n) space worst-case recursion/queue.

```java
public int numIslands(char[][] grid) {
    int m = grid.length, n = grid[0].length, count = 0;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                dfsIsland(grid, i, j);
                count++;
            }
        }
    }
    return count;
}
private void dfsIsland(char[][] g, int i, int j) {
    if (i < 0 || j < 0 || i >= g.length || j >= g[0].length || g[i][j] != '1') return;
    g[i][j] = '0';
    dfsIsland(g, i + 1, j);
    dfsIsland(g, i - 1, j);
    dfsIsland(g, i, j + 1);
    dfsIsland(g, i, j - 1);
}
```

### 42. Clone Graph
**Approach:** BFS/DFS with map from original node to cloned node.  
**Complexity:** O(V+E) time, O(V) space.

```java
public GraphNode cloneGraph(GraphNode node) {
    if (node == null) return null;
    Map<GraphNode, GraphNode> map = new HashMap<>();
    Queue<GraphNode> q = new LinkedList<>();
    q.offer(node);
    map.put(node, new GraphNode(node.val));
    while (!q.isEmpty()) {
        GraphNode cur = q.poll();
        for (GraphNode nei : cur.neighbors) {
            if (!map.containsKey(nei)) {
                map.put(nei, new GraphNode(nei.val));
                q.offer(nei);
            }
            map.get(cur).neighbors.add(map.get(nei));
        }
    }
    return map.get(node);
}
```

### 43. Course Schedule
**Approach:** Directed graph cycle detection using indegree + topological BFS (Kahn’s).  
**Complexity:** O(V+E) time, O(V+E) space.

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> g = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) g.add(new ArrayList<>());
    int[] indeg = new int[numCourses];
    for (int[] p : prerequisites) {
        g.get(p[1]).add(p[0]);
        indeg[p[0]]++;
    }
    Queue<Integer> q = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) if (indeg[i] == 0) q.offer(i);
    int done = 0;
    while (!q.isEmpty()) {
        int c = q.poll();
        done++;
        for (int nxt : g.get(c)) if (--indeg[nxt] == 0) q.offer(nxt);
    }
    return done == numCourses;
}
```

### 44. BFS / DFS Traversal
**Approach:** BFS uses queue; DFS uses recursion or stack.  
**Complexity:** O(V+E) time, O(V) space.

```java
public List<Integer> bfsTraversal(List<List<Integer>> graph, int start) {
    List<Integer> res = new ArrayList<>();
    boolean[] vis = new boolean[graph.size()];
    Queue<Integer> q = new LinkedList<>();
    q.offer(start);
    vis[start] = true;
    while (!q.isEmpty()) {
        int u = q.poll();
        res.add(u);
        for (int v : graph.get(u)) {
            if (!vis[v]) {
                vis[v] = true;
                q.offer(v);
            }
        }
    }
    return res;
}

public void dfsTraversal(List<List<Integer>> graph, int u, boolean[] vis, List<Integer> res) {
    vis[u] = true;
    res.add(u);
    for (int v : graph.get(u)) {
        if (!vis[v]) dfsTraversal(graph, v, vis, res);
    }
}
```

---

## 9) Heap / Priority Queue

### 45. Kth Largest Element in Array
**Approach:** Min-heap of size k. Keep k largest seen so far. Heap top is answer.  
**Complexity:** O(n log k) time, O(k) space.

```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    for (int n : nums) {
        pq.offer(n);
        if (pq.size() > k) pq.poll();
    }
    return pq.peek();
}
```

### 46. Top K Frequent Elements
**Approach:** Frequency map + min-heap by frequency of size k.  
**Complexity:** O(n log k) time, O(n) space.

```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.put(n, freq.getOrDefault(n, 0) + 1);
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
    for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
        pq.offer(new int[]{e.getKey(), e.getValue()});
        if (pq.size() > k) pq.poll();
    }
    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = pq.poll()[0];
    return res;
}
```

### 47. Merge K Sorted Lists
**Approach:** Put each list head in min-heap. Repeatedly pop smallest and push its next.  
**Complexity:** O(N log k) time, O(k) space.

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a.val));
    for (ListNode node : lists) if (node != null) pq.offer(node);
    ListNode dummy = new ListNode(0), cur = dummy;
    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        cur.next = node;
        cur = cur.next;
        if (node.next != null) pq.offer(node.next);
    }
    return dummy.next;
}
```

---

## 10) Dynamic Programming

### 48. Climbing Stairs
**Approach:** Fibonacci DP: ways(n) = ways(n-1) + ways(n-2).  
**Complexity:** O(n) time, O(1) space.

```java
public int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

### 49. Coin Change
**Approach:** Bottom-up DP for min coins to make every amount up to target.  
**Complexity:** O(amount * coins) time, O(amount) space.

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int a = 1; a <= amount; a++) {
        for (int c : coins) {
            if (a >= c) dp[a] = Math.min(dp[a], dp[a - c] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### 50. Longest Increasing Subsequence
**Approach:** Patience sorting idea with `tails[]`; binary search replace position.  
**Complexity:** O(n log n) time, O(n) space.

```java
public int lengthOfLIS(int[] nums) {
    int[] tails = new int[nums.length];
    int size = 0;
    for (int x : nums) {
        int l = 0, r = size;
        while (l < r) {
            int m = l + (r - l) / 2;
            if (tails[m] < x) l = m + 1;
            else r = m;
        }
        tails[l] = x;
        if (l == size) size++;
    }
    return size;
}
```

---

## Helper Data Structures

```java
class ListNode {
    int val;
    ListNode next;
    ListNode() {}
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode() {}
    TreeNode(int val) { this.val = val; }
    TreeNode(int val, TreeNode left, TreeNode right) {
        this.val = val; this.left = left; this.right = right;
    }
}

class GraphNode {
    int val;
    List<GraphNode> neighbors;
    GraphNode(int val) {
        this.val = val;
        this.neighbors = new ArrayList<>();
    }
}
```

---

## How to Use This Handbook
- Practice section-wise in order.
- For each problem: understand approach, then code without looking.
- Revisit hard problems after 2-3 days and 1 week for retention.

