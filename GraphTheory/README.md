# Graph Theory

## Path Finding
- BFS (Shortest Path)
  - Adjacency List Time Complexity O(V + E)
  - Adjacency Matrix Time Complexity O(V*V)
- DFS (All possible paths)
  - Adjacency List Time Complexity O(V + E)
  - Adjacency List Time Complexity O(V*V)
- Djikstra (Shortest single source path in a undirected graph with no negative edges)
  - Complexity V + ELogE
  - Doesn't work for -ve Edges
- Bellman Ford
  - Works for -ve Edges
  - Time Complexity O(V*V*V)
- Topological Sort
  - move items from todo list -> complete list
  - Time Complexity O(V + E) 
  - Can be used to find shortest single source path in a directed acyclic graph in that case the time complexity is O(V + E)
- Disjoint Set\Union-Find
  - Used in Kruska's Algorithm to find minimum spanning tree
  - Also to group connected nodes in a graph

