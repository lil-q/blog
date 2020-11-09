---
title: Morris 算法
date: 2020-05-01 14:51:08
tags: [algorithm]
toc: true
math: true
description: "Morris算法前序中序后序遍历二叉搜索树"
keywords: [Morris, 二叉树, 遍历, BST]
---

Morris 遍历利用的是树的叶节点左右孩子为空（树的大量空闲指针），实现空间开销的极限缩减。对于二叉树的遍历，通常采用递归或利用栈的迭代方法，两者的空间复杂度都为 $O\left ( N \right )$。Morris 遍历可以将非递归遍历中的空间复杂度降为 $O\left ( 1 \right )$。

## 前序遍历

以前序遍历为例子，比如 leecode 上的[144. 二叉树的前序遍历](https://leetcode-cn.com/problems/binary-tree-preorder-traversal/)，先给出代码：

```java
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        TreeNode cur = root; // 当前处理节点
        TreeNode tail = null; // tail记录当前节点左子树的最右节点
        while (cur != null) {
            tail = cur.left;
            if (tail != null) {
                // 找到tail的位置
                while (tail.right != null && tail.right != cur) {
                    tail = tail.right;
                }
                // tail没有连接，说明cur节点还没处理，输出这个节点
                if (tail.right == null) {
                    tail.right = cur;
                    res.add(cur.val);
                    cur = cur.left;
                    continue;
                } else { // tail连接了cur，说明已经处理完左子树，断开连接，保持树的原型
                    tail.right = null;
                }
            } else {
                res.add(cur.val);
            }
            // 左子树处理完了，开始处理右子树
            cur = cur.right;
        }
        return res;
    }
}
```

我们用 *cur* 记录正在处理的节点，用 *tail* 记录 *cur* 节点左子树的最右节点，处理 *cur* 时有三个分支：

2. `cur.left != null`，`tail.right == null`：说明 *cur* 节点及其左子树还没处理，把 *cur* 连接到 *tail*（相当于利用栈的迭代方法中的 `push()` 方法），输出这个节点，开始处理左子树 `cur = cur.left`；
2. `cur.left == null`：没有左子树，直接输出 *cur.val*，并开始处理右子树 `cur = cur.rihgt`；
3. `cur.left != null`，`tail.right == cur`：说明已经遍历完 *cur* 的左子树并返回 *cur*，断开 *tail* 的连接（利用栈的迭代方法中 `pop()` 方法），开始处理右子树 `cur = cur.rihgt`。

下图表示了前四轮循环：

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/after3.26/20200501152328.jpg)

## 中序遍历

同理我们可以实现中序遍历，类似利用栈实现中序遍历时是再出栈时输出节点，这里我们也是在处理完左子树后输出节点：

```java
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        TreeNode cur = root;
        TreeNode tail = null;
        while (cur != null) {
            tail = cur.left;
            if (tail != null) {
                while (tail.right != null && tail.right != cur) {
                    tail = tail.right;
                }
                // 这里不再输出cur
                if (tail.right == null) {
                    tail.right = cur;
                    cur = cur.left;
                    continue;
                } else {
                    res.add(cur.val); // 处理完左子树再输出，保证左中右的顺序
                    tail.right = null;
                }
            } else {
                res.add(cur.val);
            }
            cur = cur.right;
        }
        return res;
    }
}
```

注意到三个分支中两支有相同操作，我们可以简化一些代码：

```java
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        TreeNode cur = root;
        TreeNode tail = null;
        while (cur != null) {
            tail = cur.left;
            if (tail != null) {
                while (tail.right != null && tail.right != cur) {
                    tail = tail.right;
                }
                if (tail.right == null) {
                    tail.right = cur;
                    cur = cur.left;
                    continue;
                } else {
                    tail.right = null;
                }
            }
            res.add(cur.val);
            cur = cur.right;
        }
        return res;
    }
}
```

## 后序遍历

最后是后序遍历，后序遍历常用的方法是按照中右左的顺序遍历树，然后再反转输出：

```java
class Solution {
    public List<Integer> postorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        TreeNode cur = root;
        TreeNode tail = null;
        while (cur != null) {
            tail = cur.right;
            if (tail != null) {
                while (tail.left != null && tail.left != cur) {
                    tail = tail.left;
                }
                if (tail.left == null) {
                    tail.left = cur;
                    res.add(cur.val);
                    cur = cur.right;
                    continue;
                } else {
                    tail.left = null;
                }
            } else {
                res.add(cur.val);
            }
            cur = cur.left;
        }
        Collections.reverse(res);
        return res; 
    }
}
```

## 总结

利用栈的迭代方法和 Morris 遍历其实有很多相同之处，如果能结合思考就可以很快理解，Morris 算法只是把节点信息存在了某个空节点（*tail.right*）从而避免了使用栈带来的空间开销。但实际上对于每一个节点，都与要遍历去找到用于存放自己的那个空节点，Morris 算法在时间复杂度上没有优势，这是用时间换取空间。