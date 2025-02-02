---
title:  "C#_NoNeedToLearnCantonese"
date:   2025-1-31 00:53:00 +0800
tags:
  - C#
---

使用c#和ai翻译和替换文本框输入的文字-完成

<br/>

发送ai api请求的部分我使用了System.Net.Http库来模拟http请求，1月27日的记录中的问题是由于向json的模版插入提示词的参数时，参数中的换行导致json格式被打乱，导致400 bad request和404。

```csharp
using System;
using System.Net.Http;
using System.Text;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

namespace NoNeedToLearnCantonese
{
    public class AIService
    {
        private string apiKey;
        private string baseUrl;
        private string targetLanguage;
        private string content;
        private string modelName;

        public AIService(string modelName, string apiKey, string apiUrl, string targetLanguage, string content)
        {
            this.modelName = modelName;
            this.apiKey = apiKey;
            this.targetLanguage = targetLanguage;
            this.content = content.Trim();
            baseUrl = apiUrl;
        }

        public string AskAI()
        {
            if (string.IsNullOrEmpty(content))
            {
                return null;
            }

            // 使用对象构建请求体（自动处理转义）
            var requestBody = new
            {
                model = modelName,
                messages = new[]
                {
                    new
                    {
                        role = "system",
                        content = $"你是一个多语言翻译助手，请你将给定的语言翻译成{targetLanguage}，要遵循原句的标点符号。你只需要返回翻译后的句子，不需要返回其他内容。"
                    },
                    new
                    {
                        role = "user",
                        content = content
                    }
                },
                temperature = 0.3
            };

            // 序列化为 JSON（自动处理转义）
            string jsonData = JsonConvert.SerializeObject(requestBody);

            using (HttpClient client = new HttpClient())
            {
                client.DefaultRequestHeaders.Add("Authorization", $"Bearer {apiKey}");
                client.DefaultRequestHeaders.Add("Accept", "application/json");

                StringContent httpContent = new StringContent(jsonData, Encoding.UTF8, "application/json");

                try
                {
                    HttpResponseMessage response = client.PostAsync(baseUrl, httpContent).Result;
                    response.EnsureSuccessStatusCode();

                    string responseBody = response.Content.ReadAsStringAsync().Result;
                    return ExtractContent(responseBody);
                }
                catch (HttpRequestException e)
                {
                    Console.WriteLine($"Request error: {e.Message}");
                    return e.Message;
                }
            }
        }

        public string ExtractContent(string jsonResponse)
        {
            try
            {
                JObject parsedResponse = JObject.Parse(jsonResponse);
                return (string)parsedResponse["choices"][0]["message"]["content"];
            }
            catch (Exception ex)
            {
                Console.WriteLine("解析错误: " + ex.Message);
                return null;
            }
        }
    }
}
```

文本操作类：

<br/>

GetTextWithoutSelection() 获取文本框里的文本并返回

<br/>

SetTextUsingValuePattern(string newText) 替换文本框里的文本，传入的参数可以是空字符串

<br/>

```csharp
using System;
using System.Drawing;
using System.Runtime.InteropServices;
using System.Windows.Forms;
using Interop.UIAutomationClient;

namespace NoNeedToLearnCantonese
{
    public class TextOperation
    {
        public static string GetTextWithoutSelection()
        {
            // 获取文本框里的文本
            try
            {
                CUIAutomation automation = new CUIAutomation();
                IUIAutomationElement focusedElement = automation.GetFocusedElement();

                if (focusedElement == null)
                {
                    MessageBox.Show("无法获取焦点元素！");
                    return null;
                }

                // 获取 TextPattern
                object patternObj = focusedElement.GetCurrentPattern(UIA_PatternIds.UIA_TextPatternId);
                if (patternObj == null)
                {
                    MessageBox.Show("当前焦点元素不支持文本操作！");
                    return null;
                }

                IUIAutomationTextPattern textPattern = (IUIAutomationTextPattern)patternObj;

                // 直接获取全部文本内容（无需选中）
                IUIAutomationTextRange documentRange = textPattern.DocumentRange;
                string fullText = documentRange.GetText(-1); // -1 表示获取全部文本

                return fullText;
            }
            catch (Exception ex)
            {
                MessageBox.Show($"错误：{ex.Message}");
                return null;
            }
        }

        public static void SetTextUsingValuePattern(string newText)
        {
            // 替换文本框里的文本（可以替换成空字符串）
            try
            {
                CUIAutomation automation = new CUIAutomation();
                IUIAutomationElement focusedElement = automation.GetFocusedElement();

                if (focusedElement == null)
                {
                    MessageBox.Show("无法获取焦点元素！");
                    return;
                }

                object valuePatternObj = focusedElement.GetCurrentPattern(UIA_PatternIds.UIA_ValuePatternId);

                if (valuePatternObj == null)
                {
                    MessageBox.Show("当前控件不支持 ValuePattern！");
                    return;
                }

                // 安全类型转换
                IUIAutomationValuePattern valuePattern = valuePatternObj as IUIAutomationValuePattern;
                if (valuePattern == null)
                {
                    MessageBox.Show("ValuePattern 转换失败！");
                    return;
                }

                // 设置文本（防御 null 值）
                valuePattern.SetValue(newText ?? "");

                SendKeys.SendWait("{END}");
            }
            catch (Exception ex)
            {
                MessageBox.Show($"错误：{ex.Message}");
            }
     
        }
    }
}
```

一些窗口不支持直接从ui控件操作文本，我们有十分原始粗鲁的备用方案。

<br/>

GetCaretPosition() 获取文本框光标，返回System.Drawing.Point类型

<br/>

GetText() 调用了GetCaretPosition()，然后粗鲁的点击鼠标三下全选文本，复制文本，读取剪贴板

<br/>

```csharp
using System;
using System.Drawing;
using System.Windows.Forms;
using Interop.UIAutomationClient;

namespace NoNeedToLearnCantonese
{
    public class CaretDetector
    {
        // 将鼠标移动到文本框光标位置
        // 备用

        public static Point? GetCaretPosition()
        {
            try
            {
                CUIAutomation automation = new CUIAutomation();
                IUIAutomationElement focusedElement = automation.GetFocusedElement();

                // 检查焦点元素是否存在
                if (focusedElement == null)
                {
                    MessageBox.Show("无法获取焦点元素！");
                    return null;
                }

                // 获取 TextPattern
                object patternObj = focusedElement.GetCurrentPattern(UIA_PatternIds.UIA_TextPatternId);
                if (patternObj == null)
                {
                    MessageBox.Show("当前焦点元素不支持输入光标！");
                    return null;
                }

                // 转换为 TextPattern 接口
                IUIAutomationTextPattern textPattern = (IUIAutomationTextPattern)patternObj;

                // 获取选中范围
                IUIAutomationTextRangeArray selectionRanges = textPattern.GetSelection();
                if (selectionRanges.Length == 0)
                {
                    MessageBox.Show("未检测到输入光标！");
                    return null;
                }

                // 提取第一个光标范围
                IUIAutomationTextRange caretRange = selectionRanges.GetElement(0);

                // 获取光标坐标（屏幕坐标）
                double[] rects = caretRange.GetBoundingRectangles();
                if (rects != null && rects.Length >= 4)
                {
                    // 返回坐标（转换为整数像素值）
                    return new Point((int)rects[0], (int)rects[1]);
                }
                else
                {
                    MessageBox.Show("无法获取光标坐标！");
                    return null;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"错误：{ex.Message}");
                return null;
            }
        }
    }
}
```

GetText() 返回string

```csharp
[DllImport("user32.dll")]
private static extern void mouse_event(uint dwFlags, int dx, int dy, uint dwData, int dwExtraInfo);

// 定义鼠标事件常量
private const uint MOUSEEVENTF_LEFTDOWN = 0x02;
private const uint MOUSEEVENTF_LEFTUP = 0x04;

public string CopyContent()
{
    // 复制内容并读取剪贴板
    SendKeys.SendWait("^c");

    string clipboardContent = Clipboard.GetText();
    return clipboardContent;
}

public string GetText()
{
    Point? caretPos = CaretDetector.GetCaretPosition();

    if (caretPos.HasValue)
    {
        // 移动鼠标到指定坐标
        Cursor.Position = caretPos.Value;

        // 执行三次点击
        for (int i = 0; i < 3; i++)
        {
            // 发送鼠标按下事件
            mouse_event(MOUSEEVENTF_LEFTDOWN, 0, 0, 0, 0);
            // 发送鼠标释放事件
            mouse_event(MOUSEEVENTF_LEFTUP, 0, 0, 0, 0);

            // 添加短暂延迟以确保点击生效（可选）
            System.Threading.Thread.Sleep(5);
        }
    }
    else
    {
        MessageBox.Show("无法获取有效的光标位置！");
    }

    string text = CopyContent();
    SendKeys.SendWait("{BACKSPACE}");

    return text;
}
```

KeyBoardHook类：从网上抄的，完全不懂。

```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows.Forms;

namespace NoNeedToLearnCantonese
{
    class KeyboardHook
    {
        public event KeyEventHandler KeyDownEvent;
        public event KeyPressEventHandler KeyPressEvent;
        public event KeyEventHandler KeyUpEvent;

        public delegate int HookProc(int nCode, Int32 wParam, IntPtr lParam);
        static int hKeyboardHook = 0; //声明键盘钩子处理的初始值
                                        //值在Microsoft SDK的Winuser.h里查询
        public const int WH_KEYBOARD_LL = 13;   //线程键盘钩子监听鼠标消息设为2，全局键盘监听鼠标消息设为13
        HookProc KeyboardHookProcedure; //声明KeyboardHookProcedure作为HookProc类型
                                        //键盘结构
        [StructLayout(LayoutKind.Sequential)]
        public class KeyboardHookStruct
        {
            public int vkCode;  //定一个虚拟键码。该代码必须有一个价值的范围1至254
            public int scanCode; // 指定的硬件扫描码的关键
            public int flags;  // 键标志
            public int time; // 指定的时间戳记的这个讯息
            public int dwExtraInfo; // 指定额外信息相关的信息
        }
        //使用此功能，安装了一个钩子
        [DllImport("user32.dll", CharSet = CharSet.Auto, CallingConvention = CallingConvention.StdCall)]
        public static extern int SetWindowsHookEx(int idHook, HookProc lpfn, IntPtr hInstance, int threadId);


        //调用此函数卸载钩子
        [DllImport("user32.dll", CharSet = CharSet.Auto, CallingConvention = CallingConvention.StdCall)]
        public static extern bool UnhookWindowsHookEx(int idHook);


        //使用此功能，通过信息钩子继续下一个钩子
        [DllImport("user32.dll", CharSet = CharSet.Auto, CallingConvention = CallingConvention.StdCall)]
        public static extern int CallNextHookEx(int idHook, int nCode, Int32 wParam, IntPtr lParam);

        // 取得当前线程编号（线程钩子需要用到）
        [DllImport("kernel32.dll")]
        static extern int GetCurrentThreadId();

        //使用WINDOWS API函数代替获取当前实例的函数,防止钩子失效
        [DllImport("kernel32.dll")]
        public static extern IntPtr GetModuleHandle(string name);

        public void Start()
        {
            // 安装键盘钩子
            if (hKeyboardHook == 0)
            {
                KeyboardHookProcedure = new HookProc(KeyboardHookProc);
                hKeyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardHookProcedure, GetModuleHandle(System.Diagnostics.Process.GetCurrentProcess().MainModule.ModuleName), 0);
                //hKeyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardHookProcedure, Marshal.GetHINSTANCE(Assembly.GetExecutingAssembly().GetModules()[0]), 0);
                //************************************
                //键盘线程钩子
                SetWindowsHookEx(13, KeyboardHookProcedure, IntPtr.Zero, GetCurrentThreadId());//指定要监听的线程idGetCurrentThreadId(),
                                                                                                //键盘全局钩子,需要引用空间(using System.Reflection;)
                                                                                                //SetWindowsHookEx( 13,MouseHookProcedure,Marshal.GetHINSTANCE(Assembly.GetExecutingAssembly().GetModules()[0]),0);
                                                                                                //
                                                                                                //关于SetWindowsHookEx (int idHook, HookProc lpfn, IntPtr hInstance, int threadId)函数将钩子加入到钩子链表中，说明一下四个参数：
                                                                                                //idHook 钩子类型，即确定钩子监听何种消息，上面的代码中设为2，即监听键盘消息并且是线程钩子，如果是全局钩子监听键盘消息应设为13，
                                                                                                //线程钩子监听鼠标消息设为7，全局钩子监听鼠标消息设为14。lpfn 钩子子程的地址指针。如果dwThreadId参数为0 或是一个由别的进程创建的
                                                                                                //线程的标识，lpfn必须指向DLL中的钩子子程。 除此以外，lpfn可以指向当前进程的一段钩子子程代码。钩子函数的入口地址，当钩子钩到任何
                                                                                                //消息后便调用这个函数。hInstance应用程序实例的句柄。标识包含lpfn所指的子程的DLL。如果threadId 标识当前进程创建的一个线程，而且子
                                                                                                //程代码位于当前进程，hInstance必须为NULL。可以很简单的设定其为本应用程序的实例句柄。threaded 与安装的钩子子程相关联的线程的标识符
                                                                                                //如果为0，钩子子程与所有的线程关联，即为全局钩子
                                                                                                //************************************
                                                                                                //如果SetWindowsHookEx失败
                if (hKeyboardHook == 0)
                {
                    Stop();
                    throw new Exception("安装键盘钩子失败");
                }
            }
        }
        public void Stop()
        {
            bool retKeyboard = true;


            if (hKeyboardHook != 0)
            {
                retKeyboard = UnhookWindowsHookEx(hKeyboardHook);
                hKeyboardHook = 0;
            }

            if (!(retKeyboard)) throw new Exception("卸载钩子失败！");
        }
        //ToAscii职能的转换指定的虚拟键码和键盘状态的相应字符或字符
        [DllImport("user32")]
        public static extern int ToAscii(int uVirtKey, //[in] 指定虚拟关键代码进行翻译。
                                            int uScanCode, // [in] 指定的硬件扫描码的关键须翻译成英文。高阶位的这个值设定的关键，如果是（不压）
                                            byte[] lpbKeyState, // [in] 指针，以256字节数组，包含当前键盘的状态。每个元素（字节）的数组包含状态的一个关键。如果高阶位的字节是一套，关键是下跌（按下）。在低比特，如果设置表明，关键是对切换。在此功能，只有肘位的CAPS LOCK键是相关的。在切换状态的NUM个锁和滚动锁定键被忽略。
                                            byte[] lpwTransKey, // [out] 指针的缓冲区收到翻译字符或字符。
                                            int fuState); // [in] Specifies whether a menu is active. This parameter must be 1 if a menu is active, or 0 otherwise.

        //获取按键的状态
        [DllImport("user32")]
        public static extern int GetKeyboardState(byte[] pbKeyState);


        [DllImport("user32.dll", CharSet = CharSet.Auto, CallingConvention = CallingConvention.StdCall)]
        private static extern short GetKeyState(int vKey);

        private const int WM_KEYDOWN = 0x100;//KEYDOWN
        private const int WM_KEYUP = 0x101;//KEYUP
        private const int WM_SYSKEYDOWN = 0x104;//SYSKEYDOWN
        private const int WM_SYSKEYUP = 0x105;//SYSKEYUP

        private int KeyboardHookProc(int nCode, Int32 wParam, IntPtr lParam)
        {
            // 侦听键盘事件
            if ((nCode >= 0) && (KeyDownEvent != null || KeyUpEvent != null || KeyPressEvent != null))
            {
                KeyboardHookStruct MyKeyboardHookStruct = (KeyboardHookStruct)Marshal.PtrToStructure(lParam, typeof(KeyboardHookStruct));
                // raise KeyDown
                if (KeyDownEvent != null && (wParam == WM_KEYDOWN || wParam == WM_SYSKEYDOWN))
                {
                    Keys keyData = (Keys)MyKeyboardHookStruct.vkCode;
                    KeyEventArgs e = new KeyEventArgs(keyData);
                    KeyDownEvent(this, e);
                }

                //键盘按下
                if (KeyPressEvent != null && wParam == WM_KEYDOWN)
                {
                    byte[] keyState = new byte[256];
                    GetKeyboardState(keyState);

                    byte[] inBuffer = new byte[2];
                    if (ToAscii(MyKeyboardHookStruct.vkCode, MyKeyboardHookStruct.scanCode, keyState, inBuffer, MyKeyboardHookStruct.flags) == 1)
                    {
                        KeyPressEventArgs e = new KeyPressEventArgs((char)inBuffer[0]);
                        KeyPressEvent(this, e);
                    }
                }

                // 键盘抬起
                if (KeyUpEvent != null && (wParam == WM_KEYUP || wParam == WM_SYSKEYUP))
                {
                    Keys keyData = (Keys)MyKeyboardHookStruct.vkCode;
                    KeyEventArgs e = new KeyEventArgs(keyData);
                    KeyUpEvent(this, e);
                }

            }
            //如果返回1，则结束消息，这个消息到此为止，不再传递。
            //如果返回0或调用CallNextHookEx函数则消息出了这个钩子继续往下传递，也就是传给消息真正的接受者
            return CallNextHookEx(hKeyboardHook, nCode, wParam, lParam);
        }
        ~KeyboardHook()
        {
            Stop();
        }
    }
}
```

调用KeyBoardHook的示例：

```csharp
private void MainForm_Load(object sender, EventArgs e)
{
    k_hook = new KeyboardHook();
    k_hook.KeyDownEvent += K_hook_KeyDownEvent;
    k_hook.Start();
}

private void K_hook_KeyDownEvent(object sender, KeyEventArgs e)
{
    if (e.KeyCode == Keys.F2) // 可以换成其他按键
    {
        MainUtils();
    }
}
```

