# 最大流

## 基础概念
[P3376 【模板】网络最大流](https://www.luogu.com.cn/problem/P3376)
### 说明
最形象的比喻是水管流量。

核心思想是退流。

主要注意写弧优化、炸点优化。
### 代码
```cpp
#include <bits/stdc++.h>
using namespace std;

const int N = 10005;         // 根据题目调整节点数量上限
const long long INF = 1e18;  // 最大流可能爆 int，统一用 long long 比较稳妥

struct Edge {
    int to;
    long long cap;
    int rev; // 反向边在目标节点 vector 中的索引
};

vector<Edge> G[N];
int d[N];    // 分层图的深度数组 (Level Graph)
int cur[N];  // 当前弧优化数组 (Current Arc Optimization)
int n, m, s, t;

// 加边函数：同时加入正向边和反向边
void add(int u, int v, long long c) {
    G[u].push_back({v, c, (int)G[v].size()});
    G[v].push_back({u, 0, (int)G[u].size() - 1}); // 反向边初始容量为 0
}

// BFS 构建分层图，判断源汇是否连通
bool bfs() {
    memset(d, -1, sizeof(d));
    d[s] = 0;
    queue<int> q;
    q.push(s);
    
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        
        for (auto& e : G[u]) {
            if (e.cap > 0 && d[e.to] == -1) { // 还有残余容量且未被访问过
                d[e.to] = d[u] + 1;
                q.push(e.to);
            }
        }
    }
    return d[t] != -1; // 只要能到达汇点 t，就说明还有增广路
}

// DFS 多路增广寻找最大流
long long dfs(int u, long long flow) {
    if (u == t || flow == 0) return flow;
    
    long long res = 0; // 本次 DFS 在节点 u 处实际推出去的流量
    
    // 注意这里是 int& i = cur[u]，引用传递，这就是当前弧优化的核心
    // 榨干过容量的边，下次同层 DFS 就不再访问了
    for (int& i = cur[u]; i < G[u].size(); i++) {
        Edge& e = G[u][i];
        
        if (d[e.to] == d[u] + 1 && e.cap > 0) { // 必须满足严格的层级递增关系
            long long f = dfs(e.to, min(flow, e.cap));
            
            if (f > 0) {
                e.cap -= f;               // 正向边容量减少
                G[e.to][e.rev].cap += f;  // 反向边容量增加，提供反悔机制
                res += f;
                flow -= f;
                if (flow == 0) break;     // 如果当前 u 节点的可用流量已经用完，直接退出
            }
        }
    }
    return res;
}

// Dinic 主函数
long long dinic() {
    long long max_flow = 0;
    while (bfs()) { // 只要还能构建出通往 t 的分层图
        memset(cur, 0, sizeof(cur)); // 每次重新 BFS 后，当前弧都要重置
        max_flow += dfs(s, INF);
    }
    return max_flow;
}

int main() {
    ios::sync_with_stdio(0);
    cin.tie(0);
    
    // 读入节点数 n，边数 m，源点 s，汇点 t
    cin >> n >> m >> s >> t;
    
    for (int i = 1; i <= m; i++) {
        int u, v;
        long long c;
        cin >> u >> v >> c;
        add(u, v, c);
    }
    
    cout << dinic() << '\n';
    
    return 0;
}
```
## 核心建模
### 拆点法
对于限制点权，点容量的可以拆点。
常表述为点不能重复使用，不能相交。

[P2766 最长不下降子序列问题](https://www.luogu.com.cn/problem/P2766)
### DAG 最小路径覆盖 
在有向无环图（DAG）中，用最少的不相交路径覆盖所有节点，或者在规定路径条数下最多能覆盖多少个连续节点。

DAG 最小路径覆盖数 = 节点总数 - 二分图最大匹配数

直观理解：初始状态下，每个节点自己就是一条路径（共 $N$ 条）。每发生一次成功匹配（即一条边 $u \to v$ 被选中），就等于把两条路径首尾相连合并成了一条，总路径数减 1。因此，匹配数越大，合并的次数越多，最后剩下的独立路径就越少。

[P2765 魔术球问题](https://www.luogu.com.cn/problem/P2765)
## 特别技巧

### 状态分离
可以根据状态给一个点分成多个点，如按照时间分点

[P2754 [CTSC1999] 家园 / 星际转移问题](https://www.luogu.com.cn/problem/P2754)
### 动态加点与残量网络
利用网络流的残量网络性质，每次循环只把新节点加入图中，连好边后直接在原来的图上继续跑 Dinic。

之前的流不需要撤销，Dinic 会自动在残量网络上寻找新的增广路。

[P2765 魔术球问题](https://www.luogu.com.cn/problem/P2765)