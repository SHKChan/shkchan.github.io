---
title: tinyTetris源码解读
author: SHKChan
date: 2022-09-19
category: C++/C
layout: post
---
在GitHub上看到一个简单的俄罗斯方块开源项目,这里记录一下对其源码的解读.
Github地址:https://github.com/taylorconor/tinytetris

- main函数

```
int // 初值与变量最终用法无关,仅为方便打表

	 x = 431424,   // 方块x轴坐标
     y = 598356,   // 方块y轴坐标
     r = 427089,   // 某一种方块的具体那种形式 (同一个类别的方块可以经过旋转)
     px = 247872,  // 记录上一次的x 信息
     py = 799248,  // 记录上一次的y 信息
	 pr,           // 记录上一次的r 信息
     c = 348480,   // 中间变量
     p = 615696,   // 记录方块的类别， 也记录了每种方块的颜色信息,颜色信息=p+1

    tick,         // 控制帧率  控制下落的速度
    board[20][10], // 整个地图(画面),有无方块、方块的颜色信息
    // {h-1,w-1}{x0,y0}{x1,y1}{x2,y2}{x3,y3} (two bits each)
    // 方块(高,宽)和四小方块的坐标(先转二进制再按每两位读取)
    block[7][4] = { // 打表  总共有7种类别的方块  每种方块有4种旋转类型(有重复)
                    {x, y, x, y}, // 异Z型
                    {r, p, r, p}, // Z型
                    {c, c, c, c}, // O型
                    {599636, 431376, 598336, 432192}, // 异L型
                    {411985, 610832, 415808, 595540}, // L型
                    {px, py, px, py}, // I型
                    {614928, 399424, 615744, 428369} // T型
                   },
    score = 0;    // 得分

/*******************
@func: main function
*******************/
// init curses and start runloop
int main() {
  srand(time(0));
  initscr();	// 初始化curses库
  start_color();	// 初始化curses库的颜色功能
  // colours indexed by their position in the block
  for (int i = 1; i < 8; i++) {
    init_pair(i, i, 0);	// 更改某个颜色对的定义,接受参数为(要更改的颜色对编号,前景色编号,背景色编号)
  }
  new_piece();	// 创建一个新方块(未绘制)
  resizeterm(22, 22);	// 将标准窗口和当前窗口的大小调整为指定的尺寸
  noecho();	// 关闭终端回显功能
  timeout(0);	// 为窗口非阻塞读取行为,当没有输入在等待时 getch() 将返回 -1.
  curs_set(0);	// 设置光标状态为可见
  box(stdscr, 0, 0);	// 在窗口边缘绘制边框
  runloop();	// 流程主循环
  endwin();	// 关闭curses库,终端恢复正常
}
```

- NUM函数

  ```
  /*****************************
  @param  x    某种类型方块的旋转类型
  @param  y    要取的两位为y+1 和 y+2
  return  int  某两位的值
  说明：  从方块表种获取某种旋转类型的方块的特定位的值(p决定是那种方块)
  ******************************/
  int NUM(int x, int y) { return 3 & block[p][x] >> y; }
  ```
- new_piece函数

  ```
  /*******************************
  @func:  产生一个新的方块
  *******************************/
  void new_piece() {
    y = py = 0;          // 新的块,重置y和py
    p = rand() % 7;      // 块的类别是7种的随机一种
    r = pr = rand() % 4; // 旋转类型也是四种的随机一种
    x = px = rand() % (10 - NUM(r, 16)); // 17,18位   x坐标是屏幕宽度-块的宽度之间的随机数
    // NUM(r, 16)获取方块的宽度
  }
  ```
- frame函数

  ```
  /*****************************
  @func: 遍历 board 数组，在相应的位置画出对应颜色的块
  *****************************/
  void frame() {
    for (int i = 0; i < 20; i++) {
      move(1 + i, 1); // 到新的一行
      for (int j = 0; j < 10; j++) {
        board[i][j] && attron(262176 | board[i][j] << 8);
  	  // 如果board[i][j]非0 则进行后面的语句
  	  // board[i][j] << 8 等价于 COLOR_PAIR(board[i][j])
  	  // 262176实际上应为 262144 为 A_REVERSE  将前景色和背景色对换
        printw("  ");
  	  // 上一句设置输出的颜色属性，下一句关闭设置的的颜色，为下次绘制做准备
        attroff(262176 | board[i][j] << 8);
      }
    }
    move(21, 1); 	// 将光标移至 (new_y, new_x)
    printw("Score: %d", score);	// 打印得分
    refresh(); // 立即更新显示（将实际屏幕与之前的绘制/删除方法进行同步）
  }
  ```
- set_piece函数

  ```
  // set the value fo the board for a particular (x,y,r) piece
  /******************************
  @func： 给board数组(地图) 赋值， 在地图上表示出某个方块的位置信息(具体方块类型由全局变量p指定)
  @param  x  要绘制的x坐标
  @param  y  要绘制的y坐标
  @param  r  旋转的类型
  @param  v  具体的值， 非0 表示方块的颜色信息
                       0  表示擦除这个方块
  ******************************/
  void set_piece(int x, int y, int r, int v) {
    // 遍历该方块的二进制信息得到小方块坐标,再用加上(x,y)修正其坐标为board坐标系,记录颜色v
    for (int i = 0; i < 8; i += 2) { // 每个数据由2位表示，所以步进为2
      board[NUM(r, i * 2) + y][NUM(r, (i * 2) + 2) + x] = v;
    }
  }
  ```
- update_piece函数

  ```
  /********************************
  @func: 擦除旧的方块在新的位置绘制
  ********************************/
  int update_piece() {
    set_piece(px, py, pr, 0);                // 擦除原位置，在判断是否能够放下的时候要把自己影响消除，置0
    set_piece(px = x, py = y, pr = r, p + 1);// ***块的类别加1为颜色信息***， x,y为当前的新位置,并更新px,py
  }
  ```
- remove_line函数

  ```
  /********************************
  @func: 判断一行是否已经满了， 如果满了清除满的行，并得分
  ********************************/
  void remove_line() {
    for (int row = y; row <= y + NUM(r, 18); row++) {  // 遍历的范围为当前方块的顶部到底部
      c = 1; // 中间变量
      for (int i = 0; i < 10; i++) { // 遍历一行
        c *= board[row][i];
      }
      if (!c) { // 为 0 表示一行中出现了0，没有满
        continue;
      }
      for (int i = row - 1; i > 0; i--) {
        memcpy(&board[i + 1][0], &board[i][0], 40); //如果满了，就将 上一行，依次向下一行移
        // (out_write, in_read, value)
      }
      memset(&board[0][0], 0, 10); //将最顶行清空
      score++; // 得分
    }
  }
  ```
- check_hit函数

  ```
  /*********************************
  @func: 判断将方块放置在x,y位置是否会冲突
  return  int  0为无冲突
  *********************************/
  int check_hit(int x, int y, int r) {
    if (y + NUM(r, 18) > 19) { // 方块的底部已经超出了屏幕
      // NUM(r, 18)获取方块高度
      return 1;
    }
    set_piece(px, py, pr, 0);// 清空当前方块，清除自己的影响
    c = 0;
    for (int i = 0; i < 8; i += 2) {
      board[y + NUM(r, i * 2)][x + NUM(r, (i * 2) + 2)] && c++;
  	// 判断本方块要放置的四个位置 是否已经有了别的方块
    }
    set_piece(px, py, pr, p + 1);// 在原位置重新放置
    return c; // 如果c == 0 表示可以放置
  }
  ```
- do_tick函数

  ```
  /*********************************
  @func:  控制方块下落
  return  int  0为游戏结束, 1为游戏继续
  *********************************/
  int   do_tick() {
    if (++tick > 30) {
      tick = 0;
      if (check_hit(x, y + 1, r)) {
  	  // 不能放置到下一行
        if (!y) {  // y == 0 游戏结束
          return 0;
        }
        remove_line(); // 判断是否已经满了
        new_piece(); // 在开头创建新的块
      }
      else {
  	  // 可以放置到下一行
        y++;
        update_piece();
      }
    }
    return 1;
  }
  ```
- runloop函数

  ```
  /********************************
  @func: 游戏主循环，  wasd 按键处理   w: 旋转
  ********************************/
  void runloop() {
    while (do_tick()){
      usleep(10000);	// 把进程挂起一段时间， 单位是微秒（百万分之一秒）；
      if ((c = getch()) == 'a' && x > 0 && !check_hit(x - 1, y, r)) {
  	  // 左
        x--;
      }
      if (c == 'd' && x + NUM(r, 16) < 9 && !check_hit(x + 1, y, r)) {
        // 右
        x++;
      }
      if (c == 's') {
        while (!check_hit(x, y + 1, r)) { // 向下没有冲突就一直向下
          y++;
          update_piece();
        }
        remove_line();
        new_piece();
      }
      if (c == 'w') {
        ++r %= 4; // 旋转
        while (x + NUM(r, 16) > 9) {  // 横坐标超出了边界
          x--;
        }
        if (check_hit(x, y, r)) { // 如果变型后发生了碰撞，则不能变形，x还是原来的x, r还是原来的r
          x = px;
          r = pr;
        }
      }
      if (c == 'q') {
        return;
      }
      update_piece();
      frame();
    }
  }
  ```
