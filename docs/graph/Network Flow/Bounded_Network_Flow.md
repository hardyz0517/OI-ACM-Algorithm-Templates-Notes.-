# 上下界网络流
## 基础概念

在普通的网络流中，边的容量限制是 $[0, R]$。而在上下界网络流中，每条边有一个流量下界 $L$ 和上界 $R$，即要求实际流量 $f$ 满足 $L \le f \le R$。

**核心思想：分离基础流与自由流**
强行让每条边先流满下界 $L$（基础流），此时残量网络中该边的容量变为 $R - L$（自由流）。但强行塞满 $L$ 会破坏原图中各个节点的**流量守恒**，因此需要引入附加源点 $SS$ 和附加汇点 $TT$ 来“多退少补”。

引入一个差值数组 `M[i] = 节点 i 的入流下界之和 - 节点 i 的出流下界之和`。
* `M[i] > 0`：入大于出，节点 $i$ 缺水，需要从附加源点 $SS$ 补充，连边 $SS \to i$，容量为 `M[i]`。
* `M[i] < 0`：出大于入，节点 $i$ 水漫出来了，需要向附加汇点 $TT$ 排出，连边 $i \to TT$，容量为 `-M[i]`。

---
## 核心模型
### **一、 无源汇有上下界可行流**
[P14578 【模板】无源汇上下界可行流](https://www.luogu.com.cn/problem/P14578)

**模型描述**：给定一个没有源点和汇点的网络，要求每条边的流量满足 $[L, R]$，且每个点满足流量守恒（流入 = 流出）。求是否存在这样的流。

**建图与求解步骤**：
1. 建立附加源点 $SS$ 和附加汇点 $TT$。
2. 对于原图中的边 $u \to v$，下界 $L$，上界 $R$：
   * 在网络中建边 $u \to v$，容量为 $R - L$。
   * 累计 `M[v] += L`, `M[u] -= L`。
3. 遍历所有节点 $i$：
   * 如果 `M[i] > 0`，建边 $SS \to i$，容量为 `M[i]`。
   * 如果 `M[i] < 0`，建边 $i \to TT$，容量为 `-M[i]`。
4. 从 $SS$ 到 $TT$ 跑一次标准的 Dinic 最大流。
5. **判定可行性**：如果 $SS$ 连出的所有边都**满流**（即最大流等于所有 `M[i] > 0` 的 `M[i]` 之和），则存在可行流；否则无解。
6. **边的实际流量**：对应原图边在残量网络中的反向边容量（即自由流） + 下界 $L$。

**代码**：
```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
const int N = 1005;
const ll inf = 1e18;

struct Edge {
    int to; ll cap; int rev;
};
vector<Edge> g[N];
int dep[N], cur[N];
ll d[N];

struct RawEdge {
    int u, idx, low; 
};
vector<RawEdge> raw_edges;

void add_edge(int u, int v, ll w) {
    g[u].push_back({v, w, (int)g[v].size()});
    g[v].push_back({u, 0, (int)g[u].size() - 1});
}

bool bfs(int s, int t) {
    memset(dep, -1, sizeof(dep));
    dep[s] = 0;
    queue<int> q; q.push(s);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (auto &e : g[u]) {
            if (e.cap > 0 && dep[e.to] == -1) {
                dep[e.to] = dep[u] + 1;
                q.push(e.to);
            }
        }
    }
    return dep[t] != -1;
}

ll dfs(int u, int t, ll lim) {
    if (u == t || lim == 0) return lim;
    ll flow = 0;
    for (int &i = cur[u]; i < (int)g[u].size(); i++) {
        Edge &e = g[u][i];
        if (e.cap > 0 && dep[e.to] == dep[u] + 1) {
            ll f = dfs(e.to, t, min(lim, e.cap));
            if (f > 0) {
                e.cap -= f;
                g[e.to][e.rev].cap += f;
                flow += f; lim -= f;
                if (lim == 0) break;
            }
        }
    }
    if (!flow) dep[u] = -1;
    return flow;
}

ll dinic(int s, int t) {
    ll flow = 0;
    while (bfs(s, t)) {
        memset(cur, 0, sizeof(cur));
        flow += dfs(s, t, inf);
    }
    return flow;
}

int main() 
{
    cin.tie(0)->sync_with_stdio(0);
    int n, m;
    cin >> n >> m;
    int SS = 0, ST = n + 1;
    
    ll sum_pos = 0;
    for (int i = 0; i < m; i++) 
	{
        int u, v; ll l, r;
        cin >> u >> v >> l >> r;
        // 记录这条边在 g[u] 中的下标，方便后续取值
        raw_edges.push_back({u, (int)g[u].size(), (int)l});
        add_edge(u, v, r - l);
        d[u] -= l;
        d[v] += l;
    }
    
    for (int i = 1; i <= n; i++)
	{
        if (d[i] > 0) {
            add_edge(SS, i, d[i]);
            sum_pos += d[i];
        } else if (d[i] < 0) {
            add_edge(i, ST, -d[i]);
        }
    }
    
    if (dinic(SS, ST) == sum_pos)
	{
        cout << "Yes\n";
        for (auto &re : raw_edges) 
		{
            // 实际流量 = 下界 + 跑掉的自由流
            // 自由流 = 原边初始容量 - 剩余容量 = 反向边的容量
            int rev_idx = g[re.u][re.idx].rev;
            int to_node = g[re.u][re.idx].to;
            cout << re.low + g[to_node][rev_idx].cap << "\n";
        }
    } 
	else 
        cout << "No\n";
    
    return 0;
}
```

---

### **二、 有源汇有上下界最大流**
[P14579 【模板】有源汇上下界最大流](https://www.luogu.com.cn/problem/P14579)


**模型描述**：给定源点 $s$ 和汇点 $t$，每条边流量满足 $[L, R]$，求在满足可行流的前提下，$s$ 到 $t$ 的最大流量。

**建图与求解步骤**：
1. **转化为无源汇**：在原图中加入一条 $t \to s$ 的边，容量为 $\infty$（下界 0，上界 $\infty$）。这样原图的 $s$ 和 $t$ 也满足流量守恒了。
2. 按照“无源汇可行流”的方法，建立 $SS$ 和 $TT$，计算 $M$ 数组并连边。
3. 从 $SS$ 到 $TT$ 跑 Dinic 最大流。
   * 如果不满流，说明连**可行流都不存在**，直接返回无解。
4. 如果有解，此时那条 $t \to s$ 的 $\infty$ 边上的实际流量（即其反向边的容量），就是当前网络的一个**基础可行流流量**。
5. **删去附加边**：将 $t \to s$ 及其反向边从图中删去（或者将其容量清零）。注意 $SS$ 和 $TT$ 相关的边不用管，因为已经流满了。
6. **残量网络榨取**：在当前的残量网络上，直接从原源点 $s$ 到原汇点 $t$ 再跑一次 Dinic 最大流。
7. **最终结果**：有源汇上下界最大流 = **基础可行流流量 + 第二次 $s \to t$ 的最大流**。

**代码**：
```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
const int N = 1005;
const ll inf = 1e18;

struct Edge {
    int to; ll cap; int rev;
};
vector<Edge> g[N];
int dep[N], cur[N];
ll d[N];

void add_edge(int u, int v, ll w) {
    g[u].push_back({v, w, (int)g[v].size()});
    g[v].push_back({u, 0, (int)g[u].size() - 1});
}

bool bfs(int s, int t) {
    memset(dep, -1, sizeof(dep));
    dep[s] = 0;
    queue<int> q;
    q.push(s);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (auto &e : g[u]) {
            if (e.cap > 0 && dep[e.to] == -1) {
                dep[e.to] = dep[u] + 1;
                q.push(e.to);
            }
        }
    }
    return dep[t] != -1;
}

ll dfs(int u, int t, ll lim) {
    if (u == t || lim == 0) return lim;
    ll flow = 0;
    for (int &i = cur[u]; i < g[u].size(); i++) {
        Edge &e = g[u][i];
        if (e.cap > 0 && dep[e.to] == dep[u] + 1) {
            ll f = dfs(e.to, t, min(lim, e.cap));
            if (f > 0) { 
                e.cap -= f;
                g[e.to][e.rev].cap += f;
                flow += f;
                lim -= f;
                if (lim == 0) break;
            }
        }
    }
    if (!flow) dep[u] = -1;
    return flow;
}

ll dinic(int s, int t) {
    ll flow = 0;
    while (bfs(s, t)) {
        memset(cur, 0, sizeof(cur));
        flow += dfs(s, t, inf);
    }
    return flow;
}

int main() {
    cin.tie(0)->sync_with_stdio(0);
    int n, m, s, t;
    cin >> n >> m >> s >> t;
    int S = 0, T = n + 1; 
    
    ll sum = 0;
    for (int i = 0; i < m; i++) {
        int u, v; ll l, r;
        cin >> u >> v >> l >> r;
        add_edge(u, v, r - l); 
        d[u] -= l; 
        d[v] += l; 
    }
    
    for (int i = 1; i <= n; i++) {
        if (d[i] > 0) {
            add_edge(S, i, d[i]);
            sum += d[i];
        } else if (d[i] < 0) {
            add_edge(i, T, -d[i]);
        }
    }
    
    add_edge(t, s, inf); 
    int ts_edge_idx = g[t].size() - 1; // t->s 的边
    int st_edge_idx = g[s].size() - 1; // s->t 的反向边（存流量的地方）

    // 第一步：跑超级源汇最大流，看是否满足下界
    if (dinic(S, T) != sum) {
        cout << "N" << "\n"; // 对应题目的无解输出
        return 0;
    } 
    // 可行流的大小就是循环边 (t->s) 的反向边所携带的流量
    ll flow1 = g[s][st_edge_idx].cap;
    
    // 必须清零，否则退流的时候会顺着这条边流回去，导致计算错误
    g[t][ts_edge_idx].cap = 0;
    g[s][st_edge_idx].cap = 0;
    
    // 这代表尝试将已经流出的流量“退回”，尽可能压减总流量
    ll flow2 = dinic(t, s); 
    
    // 终答案是 可行流 - 退掉的流
    cout << flow1 - flow2 << "\n";
    
    return 0; 
}

/*
启示：

最小流是跑一遍可行流加退流

关于清理超级源汇：在最小流中，其实不需要像你最大流代码里那样循环清理 S, T 相关的边，只要把 T->S 那条边断掉，然后直接跑 dinic(t, s) 即可。因为 S, T 到超级源汇的边在 dinic(S, T) 满流后残量已经为 0 了，不会影响结果。 
*/
```

---

### **三、 有源汇有上下界最小流**
[P14580 【模板】有源汇上下界最小流](https://www.luogu.com.cn/problem/P14580)

**模型描述**：给定源点 $s$ 和汇点 $t$，每条边流量满足 $[L, R]$，求在满足可行流的前提下，$s$ 到 $t$ 的最小流量。

**建图与求解步骤**：
思路与最大流类似，但最后一步是“退流”。
1. 加入 $t \to s$ 容量为 $\infty$ 的边。
2. 建立 $SS$ 和 $TT$，计算 $M$ 数组连附加边。
3. 从 $SS$ 到 $TT$ 跑 Dinic。不满流则无解。
4. 记录此时 $t \to s$ 边上的流量为**基础可行流流量**。
5. 将 $t \to s$ 及其反向边拆除（容量清零）。
6. **残量网络退流**：为了让流量最小，我们需要把不需要的流量沿着原路“退”回去。直接从原汇点 $t$ 到原源点 $s$ 跑一次 Dinic 最大流。
7. **最终结果**：有源汇上下界最小流 = **基础可行流流量 - 跑出的 $t \to s$ 最大流**。

**代码**：
```cpp
#include<bits/stdc++.h>
using namespace std;

typedef long long ll;
const int N=1005;
const ll inf=1e18;

struct Edge 
{
    int to;ll cap;int rev;
};
vector<Edge> g[N]; // 全局图 g
int dep[N];        // 全局层次数组 dep
int cur[N];        // 全局当前弧优化 cur
ll d[N];           // 全局流量平衡数组

void add_edge(int u, int v, ll w) 
{
    g[u].push_back({v, w, (int)g[v].size()});
    g[v].push_back({u, 0, (int)g[u].size() - 1});
}
bool bfs(int s,int t)
{
	memset(dep,-1,sizeof(dep));
	dep[s]=0;
	queue<int>q;
	q.push(s);
	
	while(!q.empty())
	{
		int u=q.front();
		q.pop();
		for(auto & e:g[u])
		{
			if(e.cap>0 and dep[e.to]==-1)
			{
				dep[e.to]=dep[u]+1;
				q.push(e.to);
			}
		}
	}
	return dep[t]!=-1;
}

ll dfs(int u,int t,ll lim)
{
	if(u==t or lim==0)return lim;
	ll flow=0;
	for(int &i=cur[u];i<g[u].size();i++)
	{
		Edge &e=g[u][i];
		if(e.cap>0 and dep[e.to]==dep[u]+1)
		{
			ll f=dfs(e.to,t,min(lim,e.cap));
			if(d>0)
			{
				e.cap-=f;
				g[e.to][e.rev].cap+=f;
				flow+=f;
				lim-=f;
				if(lim==0)break;
			}
		}
	}
	if(!flow)dep[u]=-1;
	return flow;
}
ll dinic(int s,int t)
{
	ll flow=0;
	while(bfs(s,t))
	{
		memset(cur,0,sizeof(cur));
		flow+=dfs(s,t,inf);
	}return flow;
}
int main()
{
	cin.tie(0)->sync_with_stdio(0);
	int  n,m,s,t;
	cin>>n>>m>>s>>t;
	int S=0,T=n+1;
	
	ll sum=0;
	for(int i=0;i<m;i++)
	{
		int u,v;ll l,r;
		cin>>u>>v>>l>>r;
		add_edge(u,v,r-l);
		d[u]-=l;
		d[v]+=l;
	}
	
	for(int i=1;i<=n;i++)
	{
		if(d[i]>0)
		{
			add_edge(S,i,d[i]);
			sum+=d[i];
			/*
			d[i]>0说明强制流入大于强制流出
			那么剩下的自由流量应该流出d[i] 
			*/
		}
		else 
			add_edge(i,T,-d[i]);
	}
	//建立回流管道
	
	add_edge(t,s,inf);
	int tst = g[t].size() - 1;
	int tss = g[s].size() - 1; // 就是你代码里的 ts
	
	if(dinic(S,T)!=sum)
	{
		cout<<"N";
		return 0;
	} 
	int flow1=g[s][tss].cap;//t->s的反边，相当于t->s经过了多少流，也就是可行流 
	
	for (int i = 0; i <= n + 1; i++) //清理流 
        for (auto &e : g[i]) 
            if (i == S or i == T or e.to == S or e.to == T) e.cap = 0;
   	
	g[t][tst].cap = 0;
	g[s][tss].cap = 0;
	
	ll flow2=dinic(s,t);
	cout<<flow1+flow2<<"\n";
	return 0; 
}
```

## 例题
[P4553 80 人环游世界](https://www.luogu.com.cn/problem/P4553)

[P3980 [NOI2008] 志愿者招募](https://www.luogu.com.cn/problem/P3980)

[P3511 [POI 2010] MOS-Bridges](https://www.luogu.com.cn/problem/P3511)

[P4043 [AHOI2014/JSOI2014] 支线剧情](https://www.luogu.com.cn/problem/P4043)