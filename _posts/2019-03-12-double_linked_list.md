---
layout:     post
title:      python实现完整的双向链表并快速排序
subtitle:   
date:       2019-03-12
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - 数据结构
    - 算法
    - Python
---

## 双向链表的python实现
```python
class DoubleLinkedNode:
    """
    双向链表的节点
    """
    def __init__(self, data=None, pre=None, next=None):
        self.data = data
        self.pre = pre
        self.next = next


class DoubleLinkedList:
    """
    双向链表
    """
    def __init__(self):
        # 头节点
        self.head = DoubleLinkedNode()
        # 尾节点
        self.tail = DoubleLinkedNode()

        # 头尾相连
        self.head.next = self.tail
        self.tail.pre = self.head

    def __len__(self):
        """
        获取链表长度
        """
        length = 0
        node = self.head
        while node.next != self.tail:
            length += 1
            node = node.next
        return length

    def __reversed__(self):
        """
        反转链表
        1.node.next --> node.pre
          node.pre --> node.next
        2.head.next --> None
          tail.pre --> None
        3.head-->tail
         tail-->head
        """
        pre_head = self.head
        tail = self.tail

        def reverse(pre_node, node):
            if node:
                next_node = node.next
                node.next = pre_node
                pre_node.pre = node
                if pre_node is self.head:
                    pre_node.next = None
                if node is self.tail:
                    node.pre = None
                return reverse(node, next_node)
            else:
                self.head = tail
                self.tail = pre_head

        return reverse(self.head, self.head.next)

    def append(self, data):
        """
        添加节点
        """
        node = DoubleLinkedNode(data)
        # 找到尾部节点并将下一个节点指向新加入的节点
        pre = self.tail.pre
        pre.next = node
        node.pre = pre
        self.tail.pre = node
        node.next = self.tail
        return node

    def get(self, index):
        """
        获取第index个值，if index>0 正向获取 else 反向获取
        """
        length = len(self)
        index = index if index >= 0 else length + index
        if index >= length or index < 0: return None
        node = self.head.next
        while index:
            node = node.next
            index -= 1
        return node

    def set(self, index, data):
        """
        设置
        """
        node = self.get(index)
        if node:
            node.data = data
        return node

    def insert(self, index, data):
        """
        插入
        """
        length = len(self)
        if abs(index + 1) > length:
            return False
        index = index if index >= 0 else index + 1 + length

        next_node = self.get(index)
        if next_node:
            node = DoubleLinkedNode(data)
            pre_node = next_node.pre
            pre_node.next = node
            node.pre = pre_node
            node.next = next_node
            next_node.pre = node
            return node

    def delete(self, index):
        node = self.get(index)
        if node:
            node.pre.next = node.next
            node.next.pre = node.pre
            return True
        return False

    def clear(self):
        """
        清空
        """
        self.head.next = self.tail
        self.tail.pre = self.head

```

## 快速排序
```python
def quick_sort(head, low, high):
    """
    对双向链表的快速排序
    """
    if  head and head.next and low != high:
        p, q = low, high
        key = p.data
        while p != q:
            while p != q and q.data >= key:
                q = q.pre
            p.data = q.data
            while p != q and p.data <= key:
                p = p.next
            q.data = p.data
        p.data = key

        if low != p:
            quick_sort(head, low, p.pre)
        if p != high:
            quick_sort(head, p.next, high)
```
