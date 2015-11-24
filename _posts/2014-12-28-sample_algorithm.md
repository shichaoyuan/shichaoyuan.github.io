---
layout: post
title:  "关于sample算法"
date:   2014-12-28 22:11:14
tags: algorithm
---

# 1

最近在做的项目中有个需求，就是从热门内容库中，随机地吐出少量的热门内容。

笔者首先想到的是洗牌算法——[Fisher–Yates shuffle](http://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)。

洗牌算法顾名思义，等洗完牌后，选择前几个或者后几个就行，

因为热门内容笔者设计为不能修改的，所以实现如下：

```java
private static <T> List<T> sampleFisherYates(List<T> population, int k) {
    if (population == null) {
        return null;
    }
    
    int n = population.size();
    if (k < 0 || k > n) {
        return null;
    }
    
    Random random = new Random();
    List<T> result = new ArrayList<T>(k);
    List<T> pool = new ArrayList<T>(population);
    for (int i = 0; i < k; i++) {
        int j = random.nextInt(n-i);
        result.add(pool.get(j));
        pool.set(j, pool.get(n-i-1));
    }
    
    return result;
}
```

这个实现有个问题，如果population相对于k比较大，List的拷贝就会成为性能的瓶颈，而且还有OOM的隐患。

如何解决？设定一个阈值，如果`population.size()/k`小于某个阈值，就是使用上述方法；如果大于某个阈值，那么想其他办法。

如何设定这个阈值呢？笔者此时想起了python中有个`random.sample`方法，所以就参考一下吧——[Lib/random.py](https://hg.python.org/cpython/file/32c5b9aeee82/Lib/random.py)。

笔者简单地将python代码转成了java代码：

```java
private static <T> List<T> samplePython(List<T> population, int k) {
    if (population == null) {
        return null;
    }
    
    int n = population.size();
    if (k < 0 || k > n) {
        return null;
    }
    
    List<T> result = new ArrayList<T>(k);
    int setsize = 21;
    if (k > 5) {
        setsize += (int)Math.pow(4, Math.ceil(Math.log(k*3)/Math.log(4)));
    }
    Random random = new Random();
    if (n <= setsize) {
        List<T> pool = new ArrayList<T>(population);
        for (int i = 0; i < k; i++) {
            int j = random.nextInt(n-i);
            result.add(pool.get(j));
            pool.set(j, pool.get(n-i-1));
        }
    } else {
        Set<Integer> selected = new HashSet<Integer>();
        for (int i = 0; i < k; i++) {
            int j = random.nextInt(n);
            while(selected.contains(j)) {
                j = random.nextInt(n);
            }
            selected.add(j);
            result.add(population.get(j));
        }
    }
    return result;
}
```

基本上就是，阈值以4的指数增加，如果总量大于这个值，那么就随机的选择，如果重复了，再选一个，直到不重复。

这样基本上就满足目前的需求了。

# 2

如果热门内容库本地装不下，需要放到redis中，那么？

（当然这不太可能。。。简单计算一下，拿出1G内存对当前的机器来说是小case，一个热门内容就是几个url和几串文字，顶多500个Byte，那么能装下的热门内容条数也有两百多万。）

但是还是想一下吧，笔者在wikipedia上搜到了[Reservoir sampling](http://en.wikipedia.org/wiki/Reservoir_sampling)

```java
private static <T> List<T> sampleReservoir(List<T> population, int k) {
    if (population == null) {
        return null;
    }
    
	if (k <= 0) {
	    return null;
	}
    
    List<T> result = new ArrayList<T>(k);
    Random random = new Random();
    Iterator<T> iter = population.iterator();
    int i = 0;
    while (iter.hasNext()) {
        if (i < k) {
            result.add(iter.next());
        } else {
            int r = random.nextInt(i+1);
            if (r < k) {
                result.set(r, iter.next());
            } else {
                iter.next();
            }
        }
        i++;
    }
    return result;
}
```

对于线上环境感觉没有太合适的应用场景，如果真的数据量非常大，所有数据拉取遍历一遍，恐怕接口的响应速度就hold不住了。

所以简单地随机选，重复了就重新选就可以，当然前提是得知道redis中数据的总量。

当然对于线下，每天定时执行个hadoop任务，从一个超大集合中随机选择出一个小集合，那么修改一下上述算法还是很合适的。

另外还有选择出的集合在内存中装不下的情况，在此笔者就暂且不深究了。

# 3

还有一个很现实的需求，热门内容应该是可以设置权重的，权重高的被选择的概率大。

这个在wikipedia上没找到，所以google一下，找到了一篇论文[Weighted Random Sampling over Data Streams](http://arxiv.org/pdf/1012.0256.pdf)

其中介绍了A-Chao和A-ES两种算法，这两者的区别主要在于如何使用权重。


> The problem classes WRS-P and WRS-W differ in the way the item weights are used in the sampling procedure. In WRS-P the weights are used to directly determine the final selection probability of each item and this probability is easy to calculate. On the other hand, in WRS-W the item weights are used to determine the selection probability of each item in each step of a supposed sequential sampling procedure. In this case it is easy to study each step of the sampling procedure, but the final selection probabilities of the items seem to be hard to calculate. In the general case, a complex expression has to be evaluated in order to calculate the exact inclusion probability of each item and we are not aware of an efficient procedure to calculate this expression. An interesting feature of random samples generated with WRS-W is that they support the concept of order for the sampled items. The item that is selected first or simply has the largest key (algorithm A-ES) can be assumed to take the first position, the second largest the second position etc. The concept of order can be useful in certain applications. We illustrate the two sampling approaches in the following example.


> On-line advertisements. A search engine shows with the results of each query a set of k sponsored links that are related to the search query. If there are n sponsored links that are relevant to a query then how should the set of k links be selected? If all sponsors have paid the same amount of money then any uniform sampling algorithm without replacement can solve the problem. If however, every sponsor has a different weight than how should the k items be selected? Assuming that the k positions are equivalent in “impact”, a sponsor who has the double weight with respect to another sponsor may expect its advertisement to appear twice as often in the results. Thus, a reasonable approach would be to use algorithm A-Chao to generate a WRS-N-P of k items. If however, the advertisement slots are ordered based on their impact, for example the first slot may have the largest impact, the second the second largest etc., then algorithm A-ES may provide the appropriate solution by generating a WRS-N-W of k items.


另外如果数据量非常大，而且对性能比较关心，那么可以结合jumps方法。

论文的作者提供了算法的java代码，笔者也简单实现了一下A-ES算法：

```java
class WeightedItem<T> {
    private T value;
    private Double weight;
    public T getValue() {
        return value;
    }
    public void setValue(T value) {
        this.value = value;
    }
    public Double getWeight() {
        return weight;
    }
    public void setWeight(Double weight) {
        this.weight = weight;
    }
}

class SampledItem<T> implements Comparable<SampledItem<T>> {
    private T value;
    private Double key;
    
    public T getValue() {
        return value;
    }
    public void setValue(T value) {
        this.value = value;
    }
    public Double getKey() {
        return key;
    }
    public void setKey(Double key) {
        this.key = key;
    }

    @Override
    public int compareTo(SampledItem<T> that) {
        return this.getKey().compareTo(that.getKey());
    }
}

private static <T> List<SampledItem<T>> sampleAES(List<WeightedItem<T>> population, int k) {
    if (population == null) {
        return null;
    }
    
    if (k <= 0) {
        return null;
    }
    
    Queue<SampledItem<T>> result = new PriorityQueue<SampledItem<T>>(k);
    
    Random random = new Random();
    Iterator<WeightedItem<T>> iter = population.iterator();
    for (int i = 0; i < k; i++) {
        if (iter.hasNext()) {
            WeightedItem<T> weightedItem = iter.next();
            SampledItem<T> sampledItem = new SampledItem<T>();
            sampledItem.setKey(genKey(random, weightedItem.getWeight()));
            sampledItem.setValue(weightedItem.getValue());
            result.add(sampledItem);
        }
    }

    while (iter.hasNext()) {
        WeightedItem<T> weightedItem = iter.next();
        double key = genKey(random, weightedItem.getWeight());
        SampledItem<T> minItem = result.peek();
        if (key > minItem.getKey()) {
            result.poll();
            SampledItem<T> sampledItem = new SampledItem<T>();
            sampledItem.setKey(key);
            sampledItem.setValue(weightedItem.getValue());
            result.add(sampledItem);
        }
    }
    return new ArrayList<SampledItem<T>>(result);
}
```

[SampleUtil.java](https://github.com/shichaoyuan/snippets/blob/master/SampleUtil.java)
