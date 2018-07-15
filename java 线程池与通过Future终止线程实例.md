---
title: java 线程池与通过Future终止线程实例
date: 2015-08-02 08:53:50
tags:
---

上周在单位无聊，公司电脑又不能上网，想研究一些swing相关的东西，结果swing没怎么研究，到是写了一个比较坑爹的游戏。

游戏开始之后，出现此框，鼠标点击到此框即算过关，框框是一直在乱跳的，跳的频率随着关口的靠后会加快。

![image](https://raw.githubusercontent.com/HongkaiWen/images/master/blog/%E7%BA%BF%E7%A8%8B%E7%BB%88%E6%AD%A2/game.png)
 
设计思路：
程序加载时new一个frame，不允许frame最大化，但是鼠标滚动可以改变框框的大小（这算一后门），frame中添加一个button，在button中添加两个事件，一个是监听鼠标滚动事件，一个是监听点击事件。
点击事件发生后会改变框框跳动的频率。当累计超过10次被框框逃脱即算失败。
框框跳动的动作在一个单独的线程中进行。
 
所用技术：
1. swing
2. 多线程，线程池，线程终止

下面是完整代码：

```java
import javax.swing.*;  
import java.awt.*;  
import java.awt.event.ActionEvent;  
import java.awt.event.ActionListener;  
import java.awt.event.MouseWheelEvent;  
import java.awt.event.MouseWheelListener;  
import java.util.concurrent.*;  
  
/** 
 * Created by player on 2015/8/1. 
 */  
public class CatchMe {  
  
    private static int width = 200;  
    private static int length = 200;  
  
    private static Dimension scrSize=Toolkit.getDefaultToolkit().getScreenSize();  
  
    private static int X = (int) (scrSize.getWidth()/2);  
    private static int Y = (int) (scrSize.getHeight()/2);  
  
    private static JButton button;  
    private static JFrame frame;  
  
    private static int level = 1;  
  
    private static Future<?> task;  
  
    private static ExecutorService pool = Executors.newSingleThreadExecutor();  
  
  
    static{  
        frame = new JFrame();  
        frame.setSize(width, length);  
        frame.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);  
        frame.setLocation(X, Y);  
        frame.setTitle("catch me ~");  
        frame.setResizable(false);  
        button = new JButton();  
        button.setName("Catch Me");  
        button.setText("click to start~");  
        frame.add(button);  
    }  
  
  
    public static void main(String args[]){  
        addScrollListener();  
        addCatchListener();  
        frame.setVisible(true);  
    }  
  
    private static void addScrollListener(){  
        button.addMouseWheelListener(new MouseWheelListener(){  
            @Override  
            public void mouseWheelMoved(MouseWheelEvent event) {  
                int wheelRotation = event.getWheelRotation();  
                width += (wheelRotation * event.getScrollAmount());  
                length += (wheelRotation * event.getScrollAmount());  
                frame.setSize(width, length);  
            }  
        });  
    }  
  
    private static void addCatchListener(){  
        button.addActionListener(new ActionListener() {  
            @Override  
            public void actionPerformed(ActionEvent e) {  
                button.setText(String.format("level [%d]", level++));  
                if(task == null)  
                    task = pool.submit(moveTask);  
            }  
        });  
    }  
  
    private static void loss(){  
        //stop  
        if(task != null){  
            task.cancel(true);  
        }  
        //show  
        button.setText("mission failed at level ["+(level-1)+"]");  
        level = 1;  
        task = null;  
        width = 200;  
        length = 200;  
        frame.setSize(width, length);  
    }  
  
    private static Runnable moveTask = new Runnable() {  
        @Override  
        public void run() {  
            int escape = 0;  
            int max = 10;  
            while(!Thread.currentThread().isInterrupted()){  
                if(escape >= max){  
                    loss();  
                    escape = 0;  
                    break;  
                }  
                X = (int)(Math.random() * 1000);  
                Y = (int)(Math.random() * 800);  
                frame.setLocation(X, Y);  
                escape ++;  
                try {  
                    TimeUnit.MILLISECONDS.sleep(3000/level);  
                } catch (InterruptedException e) {}  
            }  
        }  
    };  
  
} 
```

通过ide的导出功能可以打成一个可执行的jar包，双击一下即可运行，我老婆玩了一下，说这倒霉游戏太坑爹了。

关于线程终止可以多说几句鄙人的浅显认识，java线程的终止是非抢占的，这是官方说法，大概是这么说的吧，意思是什么呢，先不管怎么能让一个线程终止，就说你让一个线程终止的时候，线程不会立刻终止，你需要在你的代码中特定的检查点检查“终止状态”，如果状态是“已经终止”线程中的代码就进行相应的处理。
 
关于“终止状态”，可以设置一个变量，比如一个布尔型的flag，默认是true，当需要终止线程的时候把这个变量编程false，那么在线程执行的代码中，比较“重要的点”，检查一下这个状态，如果编程了false就不要往下继续进行了，并可以做一些操作去响应这个终止的信号。
 
关于“重要的点”，这个要根据程序中的逻辑来判断，不可能没句话都检查一下，比如多线程中要做一个打开链接的操作，可能在打开后设置一个检查的点，如果flag变成了false，就不要往下继续进行了，并且要把链接关掉（比如jdbc链接）。（反过来想，如果java线程终止是抢占式的，那岂不是不给咱们机会去关链接了）。
 
有一种特殊情况，阻塞的代码，正在运行的代码可以通过判断状态来处理，如果是阻塞的代码，程序已经卡在那里，无法判断状态，怎么办呢（比如我上面代码中的TimeUnit.MILLISECONDS.sleep(3000/level) 这一行，程序在睡眠的时候如果想要终止程序，不可能一定要等到阻塞的代码恢复后继续走到检查点在去做判断，那样响应时间无法保证），java提供了几种方式终止线程，比如我代码中用的调用Future.cancel(true),这样代码中的阻塞语句如果发现有人调用了取消方法，就会抛出InterruptedException异常，我这里是不需要针对取消操作做什么处理的，如果是需要处理，在catch到异常后可以做善后处理。 
 
还有，我这个程序中的“终止状态”的判断用的是Thread.currentThread().isInterrupted()。

[游戏下载](https://github.com/HongkaiWen/images/blob/master/blog/%E7%BA%BF%E7%A8%8B%E7%BB%88%E6%AD%A2/catchme.jar?raw=true)