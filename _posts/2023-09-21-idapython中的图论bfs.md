# 前言

例题是minLCTF2023中的一道maze_aot。参考wp为doctor3写的。

github上可以搜的到题目和wp。

在此主要分析doctor3写的idapython脚本，从而学习idapython。

# 题目

![image-20230921175143023](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211751188.png)

![image-20230921175211489](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211752508.png)

![image-20230921175230612](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211752917.png)

![image-20230921175241371](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211752596.png)

![image-20230921175304225](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211753205.png)

![image-20230921175326335](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211753646.png)

![image-20230921175339354](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202309211753263.png)

# idapython脚本分析

我将doctor3写的wp脚本分为三个部分，每一个部分都会尽量详细地介绍。

可以通过IDA清晰地看到这个题目的难点。

第一部分脚本，我称之为生成图。

```python
#原脚本
import idaapi

function_address = 0x1500

function = idaapi.get_func(function_address)

graph = dict()
cfg = idaapi.FlowChart(function)
exclusive_nodes = list()
target_length = 65
first = None
end = 9177
jmp_dict = dict()
for block in cfg:
    graph[block.start_ea] = list()
    start_address = block.start_ea
    end_address = block.end_ea
    if first is None:
        print("Starting")
        first = start_address
    ea = 0
    if end_address - start_address <= 5:
        print("found jmp block")
        exclusive_nodes.append(start_address)
        for succ in block.succs():  # is a jmp block, ignore it
            graph[block.start_ea].append(succ.start_ea)
        continue
    flag = 0
    tgt = 0
    while (end_address - ea) != start_address:

        if idc.GetDisasm(end_address - ea).startswith("jnz"):
            # print(int(idc.GetDisasm(end_address - ea)[-4::],16))
            flag = 1
            tgt = int(idc.GetDisasm(end_address - ea)[-4::],16)
        elif idc.GetDisasm(end_address - ea).startswith("jz"):
            flag = 2
            tgt = int(idc.GetDisasm(end_address - ea)[-4::], 16)
        ea += 1
    if flag != 0:
        jmp_dict[block.start_ea] = (flag,tgt)
    for succ in block.succs():
        graph[block.start_ea].append(succ.start_ea)



print(jmp_dict)

```

```python
function_address = 0x1500

function = idaapi.get_func(function_address)
cfg = idaapi.FlowChart(function)
```

这两句话很简单，得到maze_walk这个函数的对象，并生成函数流程图。

简单来说，就是将IDA中的我们认知的流程图告诉给机器。

```python
graph = dict()
exclusive_nodes = list()
first = None
end = 9177
jmp_dict = dict()
```

graph字中键为节点，值为 这个节点后继节点的列表。

exclusive_nodes 列表用来处理jmp块，因为在流程图中jmp块单独存在，但我们知道jmp其实是跟着先前的节点。

target_length目标长度，65是需要考虑特定程序做一下变化，等到自己写程序的时候就知道了。

first是首节点。

end=9177是终点节点的定值，只不过是以10进制写出来的。

jmp_dict命名容易产生歧义，但是其目的是建立两个节点之间是如何跳转的“jnz”还是“jz"。

```python
for block in cfg:
    graph[block.start_ea] = list() #值为一个列表，存储后继节点的开始地址
    start_address = block.start_ea
    end_address = block.end_ea
    if first is None:
        print("Starting")
        first = start_address   #将首节点生成出来
    ea = 0                    # 一个变量，用来遍历这个节点的所有指令，挑选出跳转指令
    if end_address - start_address <= 5: #判断当前节点是是jmp块
        print("found jmp block")
        exclusive_nodes.append(start_address)  #将jmp块列为排除节点
        for succ in block.succs():  # is a jmp block, ignore it
            graph[block.start_ea].append(succ.start_ea) #将当前jmp块的所有后继节点加入graph 
        continue #下面的是非jmp块应该执行的内容，jmp直接continue
    flag = 0  #判断从当前节点到部分后继节点是如何过去的,jnz还是jz
    tgt = 0  #后继节点的地址
    while (end_address - ea) != start_address:   
        #看循环最后有一个ea+=1，ea的作用是用end_address - ea来索引指令

        if idc.GetDisasm(end_address - ea).startswith("jnz"):#某个指令是以jnz开头的
            # print(int(idc.GetDisasm(end_address - ea)[-4::],16))
            flag = 1   #标记为1
            tgt = int(idc.GetDisasm(end_address - ea)[-4::],16) #记录下后继节点的开始地址
        elif idc.GetDisasm(end_address - ea).startswith("jz"):#同上
            flag = 2
            tgt = int(idc.GetDisasm(end_address - ea)[-4::], 16)
        ea += 1
    if flag != 0:   # 当前节点是通过jz或jnz转移到别的节点的
        jmp_dict[block.start_ea] = (flag,tgt) #记录一下转移方式与地址
    for succ in block.succs():
        graph[block.start_ea].append(succ.start_ea) #记录后继节点的开始位置
```

总的来说，这部分作用是生成一个graph用来记录所有节点的后继节点，jmp_dict用来记录是如何到达的后继节点



第二部分脚本，是BFS跑最短路径。

```python
def BFS(grap, star):  # BFS算法
    queue = []  # 定义一个队列
    seen = set()  # 建立一个集合，集合就是用来判断该元素是不是已经出现过
    queue.append(star)  # 将任一个节点放入
    seen.add(star)  # 同上
    parent = {star: None}  # 存放parent元素
    while (len(queue) > 0):  # 当队列里还有东西时
        ver = queue.pop(0)  # 取出队头元素
        notes = grap[ver]  # 查看grep里面的key,对应的邻接点
        for q in notes:

            if q in seen:      #从这开始均为本人修改
                continue
            if q ==9177:
               
                parent[q]=ver
                seen.add(q)
                continue
            if q in exclusive_nodes:
                nex=grap[q][0]
                if nex in seen:   #这句话卡了作者两个小时
                    continue
                parent[nex]=q
                parent[q]=ver
                
                seen.add(q)
                seen.add(nex)
                queue.append(nex)
            if q not in exclusive_nodes:
               
                parent[q] = ver
                seen.add(q)
                queue.append(q)
    return parent



parent = BFS(graph, first)

p = []
a = end
while a != None:
    p.append(a)
    a = parent[a]
path = p
path=path[::-1]
```

关于BFS算法，读者可以自行百度搜索，原作者doctor3在这里的注释已经很详细了。

本人对这个BFS算法进行总结：通过BFS算法找到从起始点到所有节点的最短路径，parent字典用来记录，在最短路径的情况下，是由哪个节点（父节点）到达当前节点的。

记录完之后，再通过一个循环来找到从终点是如何到达起点的，即先找到终点的父节点，再找到终点父节点的父节点……知道找到起点。

也就是说path中的内容是逆序的。

最后得path逆序一下path=path[::-1]。

但要注意的是，如何处理jmp块，显然，我们在走到jmp时应该加入队列的是jmp块的后继块，而非jmp块。（这里原作者doctor3的wp里有问题，没有处理jmp块）（作者我调了一晚上 哭死）



第三部分，是打印路径。

```python
print(f"路径: {path}")
for i in range(len(path) - 1):
    if path[i] not in exclusive_nodes and path[i] != 5376: # 5376=0x1500 起点

        next = path[i + 1]
        if (jmp_dict[path[i]][1] == next and jmp_dict[path[i]][0] == 1):
            print('1', end="")
        elif (jmp_dict[path[i]][1] != next and jmp_dict[path[i]][0] == 1):
            print("0", end='')
        elif (jmp_dict[path[i]][1] == next and jmp_dict[path[i]][0] == 2):
            print("0", end='')
        elif (jmp_dict[path[i]][1] != next and jmp_dict[path[i]][0] == 2):
            print("1", end='')
        else:
            print("ERR")
```

第一句print是打印最短路径上每个节点的地址值。

之后这个循环，则是打印结果。

但是这里的for循环中的四个if处理了各种情况。

我们简单解说一下。

还记得我们之前的jmp_dict记录的只是jnz和jz吗？如果看到ida的流程图就知道后继块不只有jnz和jz，还有紧跟着的块。第一个if条件是，当前的节点下一步与jmp记录的意义，且是jnz跳过来的，那么就应该打印1，第二个if是，如果当前块与jmp块记录的不一样，且jmp块记录的是jnz，那么说明下一步应该是jnz相反的，也就是0。



