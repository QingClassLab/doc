# AB测试初探

### 什么是AB测试？
AB测试的概念来源于生物医学的双盲测试，双盲测试中病人被随机分成两组，在不知情的情况下分别给予安慰剂和测试用药，经过一段时间的实验后再来比较这两组病人的表现是否具有显著的差异，从而决定测试用药是否有效。互联网公司的AB测试也采用了类似的概念：将Web或App界面或流程的两个或多个版本，在同一时间维度，分别让两个或多个属性或组成成分相同（相似）的访客群组访问，收集各群组的用户体验数据和业务数据，最后分析评估出最好版本正式采用。

### AB测试的基本步骤
AB测试是一个反复迭代优化的过程，它的基本步骤如下图所示可以划分为：
1. 设定项目目标即AB测试的目标
2. 设计优化的迭代开发方案，完成新模块的开发
3. 确定实施的版本以及每个线上测试版本的分流比例
4. 按照分流比例开放线上流量进行测试
5. 收集实验数据进行有效性和效果判断
6. 根据试验结果确定发布新版本、调整分流比例继续测试或者在试验效果未达成的情况下继续优化迭代方案重新开发上线试验

本文只讨论AB测试的分流部分，一种分流算法

### 两个前提
* 不考虑灰度
* 开启AB测试后单个用户每次进入页面看到的内容相同
### 思路
1 如何才能保证分流的更准确，随机数是一个不错的思路，但差异性过大，当流量巨大时误差率<1%,由于基数过大，误差值就会很大
2 将流量分为100份，每个页面按照配置比例占用对应的份数，实际上就是一个长度为100的数组，然后再将数组打乱，这样理论上最大误差值为占比例最大的页面所占份数（因为做了打乱操作，实际误差值会比这个值低），例如：三个页面的比例：60%，30%，10%，则最大误差值为60。当流量过小时，这个误差值就不容忽视了。
表面上看，第二种方案更好一些，因为流量太小时AB测试是没有意义的，偶然因素太大会导致分析数据能反映的问题有限。
当我们单纯的考虑分流算法时，作为一个有追求的公司，这种误差也是不允许的，所以我们想出了第三种方案：
基于第二种方案，求每个页面所占份数的最大公约数，然后让份数除以最大公约数，得到一个长度更小的数组，然后再做打乱操作，这样理论上最大误差值为占比例最大的页面所占份数， 例如：三个页面的比例：60%，30%，10%，则最大误差值为6。
### 实现步骤
1. 求出参与AB测试的页面的配置比例的最大公约数
    **const** computeGreatestCommonDivisor = (numberArray) => {
      numberArray = _.compact(numberArray);
      **const** length = numberArray.length;
      **let** greatestComomonDivisor = parseInt(numberArray[0]) || 1;
      **if** (length < 2) {
        **return** greatestComomonDivisor;
      }
      **for** (**let** i = 1; i < length; i++) {
        **let** tempNumber = parseInt(numberArray[i]);
        **if** (!tempNumber) **continue**;
        **while** (tempNumber !== 0) {  欧几里得最大公约数算法
          **const** remiander = greatestComomonDivisor % tempNumber;
          greatestComomonDivisor = tempNumber;
          tempNumber = remiander;
        }
      }
      **return** greatestComomonDivisor;
    }
2. 创建一个空数组array，将每个比例除以最大公约数得到份数，放入对应份数的该页面的唯一标识，
3. 数组长度为arrayLength，将数组打乱，存入redis
4. 当用户访问时为用户生成一个本次AB测试的唯一数值index（从0开始递增），用index对arrayLength取余remainder， 保存index到redis
5. 用这个remainder作为索引从array中取值，array[remainder], 这个值就是用户访问到的页面，这样每个用户按照访问顺序，顺序的从数组中取值，当用户在此访问时取出index值做相同操作
### 几点补充
* 每组AB测试都需要为用户维护一套用户索引数据结构，在AB测试开启时重置
* 求最大公约数，初始化数组，打乱操作都是在初始化AB测试时进行的，不会影响后续的分流操作的性能
* 收集数据、分析等工作不在本文讨论范围内
