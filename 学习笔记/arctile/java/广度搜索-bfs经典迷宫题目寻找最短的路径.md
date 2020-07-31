## 广度搜索-bfs经典迷宫题目寻找最短的路径

### 题目

求解程序:迷宫以nn的0-1矩阵表示，其中1表示障碍物, 0表示可以通行;行走时的入口为(0,0) (左上角) ,出口为
(n-1,n-1) (右下角) ,行走方向为.上、下、左、右四个方向;要求输出最短的可行路径( 以数组像(1，1) (2，2)表示)，如果最短的可行路径有多条，则将它们均输出，且计算时间以毫秒为单位

### 解法

```java
package dp;

import java.io.*;
import java.util.*;


/**
 * 基本思想为尝试所有的可达可能性,然后基于递归使用回走的方式来输出过程路径
 */
public class Main {

    /**
     *  记录步行的路径,没经过一步比上一步加一
     */
    public static int isVisited[][];


    /**
     * 步数
     */
    private static int step;

    /**
     * 结果数组
     */
    private static Index[] resultSteps;

    /**
     * 行
     */
    private static  int row ;

    /**
     * 列
     */
    private static  int col;

    /**
     * 最小路径数
     */
    private static int minStep;





    public static void main(String[] args) throws IOException {
        char[][] maze = readFromFile();

        isVisited = new int[row][col];

        long start = System.currentTimeMillis();
        Queue<Index> queue = new LinkedList<>();
        //声明一个队列来保存路径
        queue.add(new Index(0, 0));
        //起点
        Index e = new Index(row-1, col-1);
        //起点为第一步
        isVisited[0][0] = 1;
        //广度搜索
        bfs(maze, queue, e);

        //打印路径
        printPath(row-1, col-1, row, col);
        long end = System.currentTimeMillis();
        System.out.println("用時為"  + (end-start));
        //System.out.println(Arrays.toString(resultSteps));

    }


    /**
     *  广度搜索
     * @param maze 迷宫声明数组
     * @param q 队列
     * @param e 终点
     */
    public static void bfs(char[][] maze, Queue<Index> q, Index e) {
        //通过声明队列来保存所有的可能下一步之后的多种可能走法
        Queue<Index> queue = new LinkedList<>();

        //当队列不为空,即还存在多种走法时,继续迭代循环
        while (!q.isEmpty()) {
            Index s = q.poll();

            //到达终点返回,循环的结束条件
            if (s.equals(e)) {
                minStep = isVisited[s.r][s.l];
                resultSteps = new Index[minStep - 1];
                return;
            }

            //判断要素主要为 :
            // 1.下一步是否出界
            // 2.是否走过
            // 3.是否是石头
 
            // 判断右边是否可走
            if (s.r + 1 <= e.r && isVisited[s.r + 1][s.l] == 0 && maze[s.r+1][s.l] == '0') {
                //可走的话将这一步加到队列中
                queue.add(new Index(s.r + 1, s.l));
                //标记已走过并且是第几步
                isVisited[s.r+1][s.l] = isVisited[s.r][s.l] + 1;
            }

            // 判断上边是否可走(感觉不太可能往上走)
            if (s.l >= 1 && isVisited[s.r][s.l - 1] == 0 && maze[s.r][s.l - 1] == '0') {
                 //queue.add(new Index(s.r, s.l - 1));
                //isVisited[s.r][s.l - 1] = isVisited[s.r][s.l] + 1;
             }

            // 判断下边是否可走
            if (s.l + 1 <= e.l && isVisited[s.r][s.l + 1] == 0 && maze[s.r][s.l + 1] == '0') {
                queue.add(new Index(s.r, s.l+1));
                isVisited[s.r][s.l + 1] = isVisited[s.r][s.l] + 1;
            }

            // 判断左边是否可走
            if (s.r >= 1 && isVisited[s.r - 1][s.l] == 0 && maze[s.r - 1][s.l] == '0') {
                queue.add(new Index(s.r - 1, s.l));
                isVisited[s.r - 1][s.l] = isVisited[s.r][s.l] + 1;
            }
        }

        //通过递归的形式来走完所有可能性的路径
        bfs(maze, queue, e);
    }


    /**
     * 理论上这里使用队列来保存多种回退的走法即可实现找到多种最短的情况
     * 使用回退的方式来记录路径
     * @param r 终点横坐标
     * @param l 终点竖坐标
     * @param n 横向一共能走多少步
     * @param m 纵向一共能走多少步
     */
    public static void printPath(int r, int l, int n, int m) {
        //到达起点返回
        if (r == 0 && l == 0) {
            return;
        }

        if (r+1 < n && isVisited[r][l] - 1 == isVisited[r+1][l]) {
            //右边可走的情况下
            printPath(r+1, l, n, m);
            stat(r, l);
        }else if (l>=1 && isVisited[r][l] - 1 == isVisited[r][l-1]) {
            //上边可走的情况
            printPath(r,l-1, n, m);
            stat(r, l);
        } else if (l +1 < m && isVisited[r][l] - 1 == isVisited[r][l+1]) {
            //下边可走的情况下(感觉不太可能往回走)
            //printPath(r, l+1, n, m);
            //stat(r,l);
        }else if (r >= 1 && isVisited[r][l] - 1 == isVisited[r-1][l]) {
            //左边可走的情况下
            printPath(r-1, l, n, m);
            stat(r, l);
        }
    }

    /**
     * 当走回到起点时,递归开始往下走进行节点输出
     * @param r
     * @param l
     */
    private static void stat(int r, int l) {
        Index index = new Index(r,l);
        resultSteps[step] = index;
        step++;
        if (step == minStep - 1){
            step = 0;
            System.out.println(Arrays.toString(resultSteps));
        }
    }

    /**
     * 从D盘读取迷宫
     */
    private static char[][]  readFromFile() throws IOException {
        File file = new File("d://Maze.txt");
        FileReader fileReader = new FileReader(file);
        BufferedReader br = new BufferedReader(fileReader);
        String line = "";
        List<String> lines = new ArrayList<>();
        while((line=br.readLine()) != null){
            lines.add(line);
        }
        br.close();
        fileReader.close();
        //获取行数
        row = lines.size();
        //获取列数
        col = lines.get(0).split(" ").length;
        //声明迷宫数组
        char[][] maze = new char[lines.size()][col];
        //遍历来生成迷宫
        for (int i = 0; i < row; i++) {
            String charline = lines.get(i).replace(" ","");
            maze[i] = charline.toCharArray();
        }
        return maze;
    }

}





/**
 * 坐标类
  */
class Index {
    int r;
    int l;
 
    public Index (int r, int l) {
        this.r = r;
        this.l = l;
    }
 
    public boolean equals(Index e) {
        return r == e.r && l == e.l;
    }

    @Override
    public String toString() {
        return "(" + r + "," + l + ")";
    }
}





```

### 参考

[蓝桥杯](https://www.liuchuo.net/archives/7047)

[letcode](https://leetcode-cn.com/problems/minimum-moves-to-reach-target-with-rotations/solution/dong-tai-gui-hua-xiang-xi-zhu-shi-jie-da-bian-yu-l/)