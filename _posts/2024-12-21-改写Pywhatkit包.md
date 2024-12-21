---
layout: post
title:  "改写Pywhatkit包的部分代码"
date:   2024-12-21 8:08:34 +0800
categories: jekyll update
---

# 不打算在C#调用python了，我把python包里的部分功能改写成C#

# 这是一个发送消息的类，定时发送的定时部分还没有改写

# 代码应该很烂，但是我也不知道烂在哪，至少能用。如果有人能指导下我就好了。



```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Diagnostics;
using System.Windows.Forms; 
using WindowsInput;
using WindowsInput.Native;

namespace WindowsFormsApp3
{
    internal class SendMessage
    {
        public string receiver { get; set; }
        public string message { get; set; }

        public SendMessage(string receiver, string message)
        {
            this.receiver = receiver;
            this.message = message;
        }

        public void Send(string receiver, string message)
        {
            OpenWeb();
            System.Threading.Thread.Sleep(7000); // 等待页面加载
            ClickMiddle();
            // 创建 InputSimulator 实例
            var sim = new InputSimulator();
            foreach (var word in message)
            {
                if (word == '\n')
                {
                    // 同时按下 Shift 和 Enter
                    sim.Keyboard.ModifiedKeyStroke(VirtualKeyCode.SHIFT, VirtualKeyCode.RETURN);
                }
                else
                {
                    // 逐个按下 message 中的每个字符
                    sim.Keyboard.TextEntry(word.ToString()); // 输入字符
                }
            }
            sim.Keyboard.KeyPress(VirtualKeyCode.RETURN);
        }

        public void OpenWeb()
        {
            // 打开浏览器输入网址
            Process.Start("https://web.whatsapp.com/send?phone="
                + this.receiver
                + "&text="
                + "(" + this.message + ")"
            );   
        }

        public void ClickMiddle()
        {
            // 获取屏幕的宽度和高度
            int screenWidth = Screen.PrimaryScreen.Bounds.Width;
            int screenHeight = Screen.PrimaryScreen.Bounds.Height;

            // 计算屏幕中心坐标
            int centerX = screenWidth / 2;
            int centerY = screenHeight / 2;

            // 移动鼠标到屏幕中心
            Cursor.Position = new System.Drawing.Point(centerX, centerY);

            // 模拟鼠标左键按下和抬起
            mouse_event(0x0002, (uint)centerX, (uint)centerY, 0, 0); // 左键按下
            mouse_event(0x0004, (uint)centerX, (uint)centerY, 0, 0); // 左键抬起
        }

        [System.Runtime.InteropServices.DllImport("user32.dll")]
        private static extern void mouse_event(uint dwFlags, uint dx, uint dy, uint dwData, int dwExtraInfo);
    }
}
```


