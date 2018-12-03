个人博客原文：[用Dijkstra算法求解无向图的最短路径](http://blog.taohuawu.club/article/29)

>Dijkstra算法是典型的算法。Dijkstra算法是很有代表性的算法。Dijkstra一般的表述通常有两种方式，一种用永久和临时标号方式，一种是用OPEN, CLOSE表的方式，这里均采用永久和临时标号的方式。注意该算法要求图中不存在负权边。 　　 　　

# **微软编程比赛里面的一道难度系数5%的编程题目如下：**

![image](http://118.126.91.74/upload/2018/04/rtgofi5vkshesphepqdbm9k68l.jpg)
![image](http://118.126.91.74/upload/2018/04/uusj6mvbfgj2kpvja0k2no03r5.jpg)

Dijkstra算法是用来求解图中顶点到另外其他顶点的最短路径的，根据题目，我们可以把每两个岛屿往来所花的最少金币当成图中的边权值，由此可以用Dijkstra算法来解决这个问题。 

![image](http://118.126.91.74/upload/2018/04/u60nao97u8h75pmdgh6gv8nhdb.jpg)

## 根据图来建立权值矩阵： 
```java
int[][] W = { { 0, 1, ５, -1, -1, -1，-1，-1 ，-1}, { 1, 0, 3, 7, 5, -1，-1，-1，-1}, { 5, 3, 0, -1, 1, 7, -1，-1，-1，-1 }, { -1, 7, -1, 0, 2, -1,3，-1，-1 }, { -1, 5, 1, 2, 0, 3, 6, 9，-1}, {-1，-1，-1, 3，6，-1, 0， 2, 7} {-1，-1，-1，-1，9，5，2, 0, 4} {-1，-1，-1，-1，-1，-1, 7, 4, 0} };
// (-1表示两边不相邻,权值无限大)
```

# 最终实现如下(java):
```java
package solution;

import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

//这个算法用来解决无向图中任意两点的最短路径  
public class Dijkstra {
  public static int dijkstra(int[][] W1, int start, int end) {  
      boolean[] isLabel = new boolean[W1[0].length];// 是否标号  
      int[] indexs = new int[W1[0].length];// 所有标号的点的下标集合，以标号的先后顺序进行存储，实际上是一个以数组表示的栈  
      int i_count = -1;//栈的顶点  
      int[] distance = W1[start].clone();// v0到各点的最短距离的初始值  
      int index = start;// 从初始点开始
      int presentShortest = 0;//当前临时最短距离  

      indexs[++i_count] = index;// 把已经标号的下标存入下标集中  
      isLabel[index] = true;  
        
      while (i_count<W1[0].length) {  
          // 第一步：标号v0,即w[0][0]找到距离v0最近的点  

          int min = Integer.MAX_VALUE;  
          for (int i = 0; i < distance.length; i++) {  
              if (!isLabel[i] && distance[i] != -1 && i != index) {  
                  // 如果到这个点有边,并且没有被标号  
                  if (distance[i] < min) {  
                      min = distance[i];  
                      index = i;// 把下标改为当前下标  
                  }  
              }  
          }  
          if (index == end) {//已经找到当前点了，就结束程序  
              break;  
          }  
          isLabel[index] = true;//对点进行标号  
          indexs[++i_count] = index;// 把已经标号的下标存入下标集中  
          if (W1[indexs[i_count - 1]][index] == -1 
                  || presentShortest + W1[indexs[i_count - 1]][index] > distance[index]) {  
              // 如果两个点没有直接相连，或者两个点的路径大于最短路径  
              presentShortest = distance[index];  
          } else {  
              presentShortest += W1[indexs[i_count - 1]][index];  
          }  

          // 第二步：将distance中的距离加入vi  
          for (int i = 0; i < distance.length; i++) {  
              // 如果vi到那个点有边，则v0到后面点的距离加  
              if (distance[i] == -1 && W1[index][i] != -1) {// 如果以前不可达，则现在可达了  
                  distance[i] = presentShortest + W1[index][i];  
              } else if (W1[index][i] != -1 
                      && presentShortest + W1[index][i] < distance[i]) {  
                  // 如果以前可达，但现在的路径比以前更短，则更换成更短的路径  
                  distance[i] = presentShortest + W1[index][i];  
              }  

          }  
      }  
      //如果全部点都遍历完，则distance中存储的是开始点到各个点的最短路径  
      return distance[end] - distance[start];  
  }  
  private static class Island{
	  public int x,y;
  }
  public static void main(String[] args) {  
	  ArrayList<Island> arr=new ArrayList<Island>();
      // 建立一个权值矩阵  
      
      int [][] Test={
    		  {0,1,4},{1,0,1},{4,1,0}
      };
      Scanner input=new Scanner(System.in);
      int num=input.nextInt();
      for (int i = 0; i < num; i++) {
    	  Island island=new Island();
    	  island.x=input.nextInt();
    	  island.y=input.nextInt();
          arr.add(island);
	}
      int [][] t=new int[num][num];
      for (int i = 0; i < t.length; i++) {
    	  for (int j = 0; j < t.length; j++) {
			t[i][j]=Math.min(Math.abs(arr.get(i).x-arr.get(j).x), Math.abs(arr.get(i).y-arr.get(j).y));
		}
		
	}
      System.out.println(dijkstra(t, 0,num-1)); 
  }  
}
```
