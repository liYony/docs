# LWP中数据结构和API介绍

## 1 lwp_avl

### 1.1 数据结构

#### lwp_avl_struct

```c
struct lwp_avl_struct
{
    struct lwp_avl_struct *avl_left;
    struct lwp_avl_struct *avl_right;
    int    avl_height;
    avl_key_t avl_key;
    void *data;
};
```

- avl_left：平衡二叉树节点的左子树节点。
- avl_right：平衡二叉树节点的右子树节点。
- avl_height：平衡二叉树节点的高度。
- avl_key：平衡二叉树节点的键值。
- data：平衡二叉树节点的私有数据。

### 1.2 API介绍

```c
void lwp_avl_remove(struct lwp_avl_struct * node_to_delete, struct lwp_avl_struct ** ptree);
void lwp_avl_insert (struct lwp_avl_struct * new_node, struct lwp_avl_struct ** ptree);
struct lwp_avl_struct* lwp_avl_find(avl_key_t key, struct lwp_avl_struct* ptree);
int lwp_avl_traversal(struct lwp_avl_struct* ptree, int (*fun)(struct lwp_avl_struct*, void *), void *arg);
struct lwp_avl_struct* lwp_map_find_first(struct lwp_avl_struct* ptree);
```

#### lwp_avl_remove

主要完成平衡二叉树中节点的删除。

```c
void lwp_avl_remove(struct lwp_avl_struct * node_to_delete, struct lwp_avl_struct ** ptree);
```

| **参数**       | **描述**               |
| -------------- | ---------------------- |
| node_to_delete | 需要删除的二叉树节点。 |
| ptree          | 平衡二叉树的根节点。   |
| **返回**       | 无                     |

#### lwp_avl_insert

主要完成平衡二叉树中节点的插入。

```c
void lwp_avl_insert (struct lwp_avl_struct * new_node, struct lwp_avl_struct ** ptree);
```

| **参数**       | **描述**               |
| -------------- | ---------------------- |
| node_to_delete | 需要插入的二叉树节点。 |
| ptree          | 平衡二叉树的根节点。   |
| **返回**       | 无                     |

#### lwp_avl_find

根据节点的key键值，在二叉树中寻找对应的节点。

```c
struct lwp_avl_struct* lwp_avl_find(avl_key_t key, struct lwp_avl_struct* ptree);
```

| **参数**   | **描述**                     |
| ---------- | ---------------------------- |
| key        | 需要查找的二叉树节点的键值。 |
| ptree      | 平衡二叉树的根节点。         |
| **返回**   | ——                           |
| RT_NULL    | 失败。                       |
| 二叉树节点 | 成功。                       |

#### lwp_avl_traversal

用于遍历AVL树的函数。它采用了递归的方式来遍历树的每一个节点，并对每个节点执行一个由调用者提供的函数 fun。这个函数接受一个AVL树节点（ptree）、一个函数指针（fun）以及一个额外的参数（arg）。

```c
int lwp_avl_traversal(struct lwp_avl_struct* ptree, int (*fun)(struct lwp_avl_struct*, void *), void *arg);
```

| **参数** | **描述**                   |
| -------- | -------------------------- |
| ptree    | 平衡二叉树的根节点。       |
| fun      | 由调用者提供的函数。       |
| arg      | 由调用者提供的函数的参数。 |
| **返回** | ——                         |
| 0        | 完全遍历完所有节点。       |
| 非0      | 可能未完全遍历完所有节点。 |

> [!NOTE]
>
> lwp_avl_traversal 函数的返回值是一个整数，这个整数是由调用者提供的函数 fun 决定的。具体来说，当 fun 被应用到每个节点时，它返回一个整数。lwp_avl_traversal 函数在遍历过程中会检查 fun 的返回值：
>
> - 如果 fun 在任何时候返回非0值，lwp_avl_traversal 将立即停止遍历，并返回该非0值。
> - 如果 fun 对所有节点的调用都返回0，那么 lwp_avl_traversal 函数在完成遍历后也会返回0。
>
> 这种设计使得 lwp_avl_traversal 函数非常灵活，因为它可以根据 fun 的返回值来定制遍历的行为。例如：
>
> - 如果 fun 是一个检查函数，用于查找具有特定属性的节点，并且当找到这样的节点时返回非0值，那么 lwp_avl_traversal 可以用于在树中搜索特定的节点。
> - 如果 fun 是一个修改函数，用于更新节点的数据，并且当某个更新失败时返回非0值，那么 lwp_avl_traversal 可以用于遍历树并应用更新，同时在第一次失败时停止遍历。
> - 如果 fun 是一个简单的打印函数，用于打印节点的信息，并且总是返回0，那么 lwp_avl_traversal 可以用于打印整棵树的内容。
>
> 总之，lwp_avl_traversal 的返回值是依赖于 fun 的实现和它在遍历过程中的行为的。它提供了一种机制，使得调用者可以根据需要在遍历过程中定制和控制行为。

#### lwp_map_find_first

根据节点的key键值，在AVL树中查找并返回最左侧的节点（即最小键值的节点）。

```c
struct lwp_avl_struct* lwp_map_find_first(struct lwp_avl_struct* ptree);
```

| **参数**   | **描述**             |
| ---------- | -------------------- |
| ptree      | 平衡二叉树的根节点。 |
| **返回**   | ——                   |
| RT_NULL    | 失败。               |
| 二叉树节点 | 成功。               |

