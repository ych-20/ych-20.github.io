# Vue 中的 Diff算法
---

## 2.x的Diff算法

```
diff算法是用于父节点相同，对子节点更新的一种算法。
一下说明全部以如下情况为例子：   
真实DOM： A | B | C | D | E | F | G 
old   ： A | B | C | D | E | F | G 
new   ： A | B | D | E | I | C | F | G
```

### 比较步骤

`通过双指针从两头往中间遍历：newStartIdx（oldStartIdx） 和 newEndIdx（oldEndIdx）`

* 新前 和 旧前比较
    - new A 和 old A 进行比较，相同则 newStartIdx++、oldStartIdx++。B同理
* 新后 和 旧后比较
    - new G 和 old G 进行比较，相同则 newEndIdx--、oldEndIdx--。F同理
* 新前 和 旧后比较
    - 此时各个指针指向newStartIdx：D、newEndIdx：C、oldStartIdx：C、oldEndIdx：E
    - D 和 E 对比 不相同 进行下一步对比
* 新后 和 旧前比较
    - C 和 C 对比 相同 则 将真实DOM中C的位置移动到 oldEndIdx 的后面。 然后将 oldStartIdx++、newEndIdx--
    - 此时各个指针指向newStartIdx：D、newEndIdx：I、oldStartIdx：D、oldEndIdx：E
    - 继续newStartIdx开始重复 上述步骤
* 遍历旧数组进行查找(isSameNode) 如果找到 则进行移动 未找到则新增当前节点。
    - 如果节点经过上述四种情况都没匹配上，则去遍历old 看是否 有节点和 newStartIdx 匹配，根据结果进行响应的 移动、删除、新增DOM操作
* old先遍历完成 则将new中未遍历的节点 插入到真实DOM中。反之则删除odl未处理的节点对应的DOM节点

## Vue 3.x 中对于Diff 算法的优化

1. 分别从头尾开始遍历，找到第一个不匹配的项则停止进行下一步匹配。
    - 从前往后匹配到B后结束 从后往前 匹配到F结束。这四个节点不需要移动
    - 旧节点剩下未遍历DOM：C | D | E
    - 新节点剩下未遍历DOM：D | E | I | C

3. 判断old是否遍历完，遍历完则将new中剩下的全部新增（mount） 

4. 判断new是否遍历完，遍历完则将old中剩下的全部删除（unmount） 

5. 剩下的中间内容为乱序
    - 根据new剩下的节点生成一个key和index的map，keyToNewIndexMap[D:3,E:4,I:5,C:6]（5.1）
    - 根据上述map生成一个等长的数组 newIndexToOldIndexMap[ 0, 0, 0, 0]（5.2）
    - 遍历old去匹配keyToNewIndexMap 如果匹配到 则把 oldIdx + 1填充到新数组（newIndexToOldIndexMap）对应的地方去，
    未匹配到的节点就是需要删除的节点。找到了patch一下，最后 newIndexToOldIndexMap 为 [ 4, 5, 0, 3]。（5.3）
    - 其实到这一步，已经很好办了，从尾到头循环一下newIndexToOldIndexMap
    是 0 的，说明是新增的数据，就 mount 进去
    非 0 的，说明在旧数据里，我们只要把它们移动到对应index的前面就行了
    但是这里还可以做一步优化（根据最长递增子序列 进行优化，即找到最长递增子序列[ 4, 5]不用移动 只需要新增0[I 到 DE后面]和移动3[C到I后]）
    
