# NC78 反转链表

**方法1：**递归法

```kotlin
fun ReverseList(head: ListNode?): ListNode?  {
    if(head == null || head.next == null) return head // 尾指针或空列表则直接返回
    val nextList = head.next!! // 指向 L(i+1)
    val newHeadList = ReverseList(nextList) // 对 {L(i+1)} 列表进行逆序
    nextList.next = head // 将 L(i+1) 指向 L(i) 
    head.next = null // 将 L(i) 指向一个空地址 至此实现 L(i+1) 和 L(i) 的反转
    return newHeadList
}
```

**方法2：**遍历法

