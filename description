package com.demo;

public class Demo {

	public static void main(String[] args) {
		long t = System.currentTimeMillis();
		for (int i = 0; i < 100000; i++) {
			String s0 = String.valueOf(11);
		}
		System.out.println("耗时" + (System.currentTimeMillis() - t));//耗时15

		t = System.currentTimeMillis();
		for (int i = 0; i < 100000; i++) {
			String s = "" + 11;
		}
		System.out.println("耗时" + (System.currentTimeMillis() - t));//耗时1

		String str = "";
		t = System.currentTimeMillis();
		for (int i = 0; i < 100000; i++) {
			String s = str + 11;
		}
		System.out.println("耗时" + (System.currentTimeMillis() - t));//耗时25

		t = System.currentTimeMillis();
		StringBuilder builder = new StringBuilder();// 循环中最好这样
		for (int i = 0; i < 100000; i++) {
			String s = builder.append(11).toString();
		}
		System.out.println("耗时" + (System.currentTimeMillis() - t));//耗时3873
	}

}
