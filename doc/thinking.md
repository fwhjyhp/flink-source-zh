前面已经写过了太多的Flink源码分析的东西，可能讲得比较偏理论，今天呢，就来分享一个实际工作中碰到的Flink场景下的问题，也是自己编码过程中的一个教训。
先来简单描述一下场景，实际的场景比较复杂，这里就简单的抽象一下。

在电商业务中，如果我们需要对每一笔订单进行统计来计算销售额，比如说，618和双十一时的订单金额统计。对于每一个用户来说，他可能先在电商网站上下一个单，
这个订单会包含多个所要购买的商品，此时就需要将其金额加入到实时销售额里面，而我们同时可以对这个订单进行不同的操作，比如修改物流地址、取消订单等。那么，
在电商大促实时大屏上就应该根据我们对其的实际操作增减销售额，这就需要保证对每个订单的顺序才能保证处理的准确性，如果取消订单的消息的处理早于下单和修改
物流地址，那么最终的处理结果就可能不正确。那么如何保证实时数据处理的可靠性？

假设我们使用的是kafka接入订单流，然后进行处理，并且处理完成的消息写回kafka，其实就变成了两个问题：如何保证kafka接入数据的顺序、如何保证flink处理的
顺序？

首先，如何保证kafka接入数据的顺序？emmmmmm....这是个在实时计算面试中经常会问的问题，但是请一定注意，这是个陷阱问题，因为kafka根本就无法保证全局
消息的有序性，在它的各个分区间的消息一定是无法保证顺序的，但是kafka各个分区内的数据是有序的，那么我们就可以利用上这个特性。如果我们在使用kafka接
入数据时，将相同订单的消息放到同一分区，那么就能保证订单的创建消息一定早于订单的取消的消息，即第一个问题解决。

在解答第二个问题之前，先要说一下，此处的乱序与flink的EventTime事件乱序并不是很一样，在此处由于数据接入到kafka的分区时保证了相同订单的消息的有序，
所以只需要保证相同的订单的消息能够被同样的flink消费线程消费，并且在整个flink的处理流程中，所有相同的订单的消息也都能被每个算子的相同子分区处理即可。
如果算子之间的流分区是forward、global、hash或是上下游算子chain在一起都是可以保证相同订单的消息的处理顺序，否则就需要进行自定义分区，确保相同订单
的消息被分配到相同的分区处理。

终于说到了重点，在实现自定义分区时，可以通过Partitioner实例的自定义partition方法将记录输出到下游，代码如下：
new Partitioner<String>(){
    @Override
    public int partition(String key, int numPartitions){
        int hashCode = key.hashCode();
        int result = hashCode % numPartitions;
        return result;
    }
}

在上面的代码中，将订单id作为key传入，获取其hash值，而numPartitions会自动赋值为下游算子的并行度，取模后所得的值表示的是数据被传递到下游算子的哪一个
分区中。将上面的这个类的实例传给partitionCustom()方法作为参数，一个简单的自定义分区器就完成了。看上去很easy，对不对？

满心欢喜的进行测试，发现会时不时的报分区取值不能为负值，然后程序就会自动重启，导致消息处理失败。这就奇怪了，哪里会有负值呢？由于报错信息并不明确，所以
起初以为自定义分区实现方式不对或是改动的其它地方产生了什么影响，但是也没找出个中原因，于是删掉上述自定义分区代码，发现报错没有了，确定是自定义分区代码
问题。但是这个代码这么简单，貌似也看不出什么问题来，偶然间想到java的取模计算的一个问题，就是取模的结果的符号与被模数一致，而不是自动变成正数，也就是
-10%3=-1，而10%(-3)=1，这与很多其它语言的实现不太一样，但是由于并不经常碰到这种场景，所以一直将其忘记了。于是将代码调整为一下：
new Partitioner<String>(){
    @Override
    public int partition(String key, int numPartitions){
        int hashCode = key.hashCode();
        int hashCodeAbs = Math.abs(hashCode);
        int result = hashCodeAbs % numPartitions;
        return result;
    }
}

心想这下应该没问题了吧，测试环境测试貌似也很成功，跑了两天没出问题，于是上线到线上，本以为事情就这么顺利的解决了，没想到过了半个月左右，发现程序又突然间
重启，导致数据的处理出现积压，看异常还是分区不能为负数，这就奇怪了，不是已经解决了吗，而且跑了这么久也没出过问题，怎么突然就有问题了，看线上版本也没有被
人修改，反编译代码发现先前的调整也在，这说明就是这个现有的代码还是有问题，可是查看代码，hash结果已经经过了取绝对值，这结果应该肯定是正数或0才对，不应该
冒出一个负值。百思不得其解，让人无比郁闷！没办法，只能加上调试代码，将result在负值时的key取值和hashCode及hashCodeAbs全部打印出来，发现hashCode和
hashCodeAbs的取值都是Integer.MIN_VALUE，于是写程序简单验证Math.abs(Integer.MIN_VALUE)的结果，居然真的是Integer.MIN_VALUE，于是点进去
Math.abs()方法的源码，发现这样一段话：注意，如果参数的值为Integer.MIN_VALUE，这个整型数据所能表示最小的负值，那么将返回原值，且是一个负值。其实很好
理解，Integer.MIN_VALUE的取值是-2147483648，其取绝对值就是+2147483648，但一个32位整数可以表示的最大值是+2147483647，+2147483648超出了范围。于是
被"翻转"成了-2147483648，也就是原值。问题终于找到了答案，于是再次将代码调整为：
new Partitioner<String>(){
    @Override
    public int partition(String key, int numPartitions){
        int hashCode = key.hashCode();
        int hashCodeAbs = Math.abs(hashCode);
        int result = hashCodeAbs % numPartitions;
        return (result < 0 ? (result + numPartitions) : result);
    }
}

这下终于彻底将这个问题解决，再也没有出现问题了。