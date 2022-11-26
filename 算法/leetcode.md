#### [无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)

双指针，右指针先走，用hash的方法统计字符串，遇到第一个重复的字符就开始移动左指针，直到字串中没有重复的字符为止。

```java
import java.util.*;

class Solution {
    public int lengthOfLongestSubstring(String s) {
        if(s == null || "".equals(s)) return 0;
        
        int[] hash = new int[256];
        int left = 0;
        int right = 0;
        int maxLen = Integer.MIN_VALUE;
        int slen = s.length();
        while(right < slen) {
            char rightChar = s.charAt(right);
            hash[rightChar]++;
            while(hash[rightChar] > 1) {
                char leftChar = s.charAt(left);
                hash[leftChar]--; // -1是为了把右指针指向的字符给去掉，跳出循环。
                left++;
            }
            maxLen = Math.max(maxLen, (right - left + 1));
            right++;
        }
        return maxLen;
    }
}
```

#### [25. K 个一组翻转链表](https://leetcode.cn/problems/reverse-nodes-in-k-group/)

递归思想，先反转前k个，后续的走一样的流程。

重点：找分割节点，反转后指针的交换。

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
       if(k == 0 || head == null) return null;
        ListNode pre = head;
        ListNode cur = head;
        // 查找分割节点
        for(int i = 0; i < k - 1 && cur != null; i++){
            cur = cur.next;
        }
        if (cur == null) {
            return head;
        }
        
        ListNode next = cur.next;
        cur.next = null;
        reverList(pre);

        // cur和head节点交换
        ListNode temp = cur;
        cur = head;
        head = temp;

        ListNode newHead = reverseKGroup(next, k);
        cur.next = newHead;
        return head;
    }

    public ListNode reverList(ListNode head){
        // 返回最后的一个节点
        if(head == null || head.next == null){
            return head;
        }
        ListNode next = head.next;
        head.next = null;
        ListNode newHead = reverList(next);
        next.next = head;
        return newHead;
    }
}
```

