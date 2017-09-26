---
layout: post
title: Brust Balloons
category: project
description: LeetCode Brust Balloons
---
# [{{ page.title }}][1]
{{ page.date | date_to_string }} By {{ site.author_info }}


[Edward]:    http://Edward0205.github.io  "Edward"
[1]:    {{ page.url}}  ({{ page.title }})

题目如下:

Given n balloons, indexed from 0 to n-1. Each balloon is painted with a number on it represented by array nums. You are asked to burst all the balloons. If the you burst balloon i you will get nums[left] * nums[i] * nums[right] coins. Here left and right are adjacent indices of i. After the burst, the left and right then becomes adjacent.

Find the maximum coins you can collect by bursting the balloons wisely.

Note: 
(1) You may imagine nums[-1] = nums[n] = 1. They are not real therefore you can not burst them.
(2) 0 ≤ n ≤ 500, 0 ≤ nums[i] ≤ 100

乍一看，这个题目可以应该可以通过暴力解法去解出答案，但是相应的时间复杂度为O(n!)，按照提出给出的n的范围，在0-500之间，这个时间复杂度应该是不能够满足要求的。

思考一下其实从这个题目里面隐约可以看出一些"重复子问题"的影子，如果我们每次Brust掉一个气球了，我们获取到Brust这个气球的Coins和两个气球集合的子集合(这里就叫左集合和右集合)，我们能不能找到一个合适的切入点，来对这个问题用DP去求解呢？

我们选择Brust掉一个气球，我们如何去计算出他的Coins？实际上只有两种情况能够计算出这个Coins：

一、这个气球是我们第一个Brust掉的气球，Coins值为:nums[i-1] * nums[i] * nums[i+1]

二、这个气球使我们最后一个Brust掉的气球，Coins为:1 * nums[i] * 1

如果这个气球不是在这两种情况里面的，我们是计算不出来的，因为我们不知道在某个情况下这个气球的邻居是谁

另外在DP问题里面，我们要注意到无后效性，如果我们选择第一种情况，是不满足无后效性的，只有第二种情况才能满足无后效性，因为我们Brust掉的是最后一个气球了，当然是不会对后面的结果产生任何影响了。

因此我们从第二种情况切入，便可以将这个问题以DP来求解

DP问题的最优解是：Max([left,right]中最后Brust掉 : 第i个气球的Coins值+左子集的最优值+右子集的最优值)

参考解法:

    public int maxCoins(int[] iNums) {
    	int[] nums = new int[iNums.length + 2];
    	int n = 1;
    	for (int x : iNums) if (x > 0) nums[n++] = x;
    	nums[0] = nums[n++] = 1;


    	int[][] dp = new int[n][n];
    	for (int k = 2; k < n; ++k)
        	for (int left = 0; left < n - k; ++left) {
            	int right = left + k;
            	for (int i = left + 1; i < right; ++i)
                	dp[left][right] = Math.max(dp[left][right], 
                	nums[left] * nums[i] * nums[right] + dp[left][i] + dp[i][right]);
        	}

    	return dp[0][n - 1];
    }


这里有个需要关心的地方就是，解法里面将所有num[n]=0的值都先去掉了，因为Brust一个0，不会得到任何的Coins，同时在数组的首尾放上了两个哨兵，来方便处理边界问题.