---
title: java中的回调函数
date: 2015-06-18 18:15:42 +0800
toc: false
categories: [ java ]
tags: [ java ]
---

最近学习内部类的时候，对Java实现回调函数机制有了进一步了解，自己整理点比较，希望大家可以相互讨论。
所谓回调，就是允许客户类通过内部类引用来调用其外部类的方法，这是一种非常灵活的功能。

由于java暂时还不能显示支持闭包(Closure)，不过听说新版可以支持了，不过我还没用过。现在暂时用的是非静态内部类实现回调功能。
<!-- more -->

**情形一**

假设有一个老师Teacher对象，平时的工作是上课，周末的工作在家干农活（乡村老师大部分都这样），方法名都是work，但功能都不一样，可以用内部类实现这种需求：

```java
public class Teacher {
	// 正常的工作
	public void work() {
		System.out.println("平常我在给学生教课");
	}

	// 业余的工作
	public void farming() {
		System.out.println("周末我在农田忙活");
	}

	private class Farmer {
		// 非静态内部类回调外部类实现work方法，
                // 非静态内部类引用的作用仅仅是向客户提供一个回调外部类的途径
		public void work() {
			farming();
		}
	}

	public Farmer getCallbackReference() {
		return new Farmer();
	}

	public static void main(String[] args) {
		Teacher t = new Teacher();
		// 直接调用work
		t.work();
		// 表面上调用的是Farmer的work方法，实际上是回调Teacher的farming方法
		t.getCallbackReference().work();
	}
}
```

**情形二**

Swing中响应按钮点击事件，使用匿名内部类，各个不同的控件发生事件后可以回调外部类中对应的处理方法。

```java
public class ButtonFrame extends JFrame {
	// 红色按钮
	private JButton redButton = new JButton("Red Button");
	// 蓝色按钮
	private JButton blueButton = new JButton("Blue Button");

	// 处理红色按钮的方法
	private void processRedButton() {
		System.out.println("红色按钮被点击了");
	}

	// 处理蓝色按钮的方法
	private void processBlueButton() {
		System.out.println("蓝色按钮被点击了");
	}

	public ButtonFrame() {
		setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
		redButton.addActionListener(new ActionListener() {
			public void actionPerformed(ActionEvent e) {
				// 回调ButtonFrame中处理红色按钮的方法
				processRedButton();
			}
		});
		blueButton.addActionListener(new ActionListener() {
			public void actionPerformed(ActionEvent e) {
				// 回调ButtonFrame中处理蓝色按钮的方法
				processBlueButton();
			}
		});
		getContentPane().add(redButton);
		getContentPane().add(redButton);
	}

	public static void main(String[] args) {
		new ButtonFrame().setVisible(true);
	}
}
```

最后，顺便提一下C语言中实现这种回调的机制，就是利用函数指针的方式实现的。在《Pointers On C》这本书里面举了一个例子很详细的说明这个问题：

要编写一个类型无关的函数，在一个链表中查找一个指定的值，这个链表的数据元素可以是整数，也可能是字符串等，所以函数的参数就不能特定类型，C语言里面有个void类型，可以指向任何类型，通过这个，可以编写一个与类型无关的链表查找函数：

```c
/**
* 在一个单链表中查找一个指定值的函数。它的参数是一个指向链表第1个节点的指针，
* 一个指向我们需要查找的指针和一个函数的指针，它所指向的函数用于比较存储于链表
* 中的类型的值
*
*/

#include ;
#include ;

/*类型无关的链表查找*/
Node * search_list() (Node *node, void const *value,
	int (*compare) (void const *, void const *)) {
		while (node != NULL) {
			if (compare( &node->value, value) == 0) {
				break;
			}
			node = node->link;
		}
		return node;
	}
```
