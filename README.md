#架构
1. 服务器端是由：python中的Rest API接口

2. 前端是由：angularjs、bootstrap、html5

3. 移动端是由：ionic

#功能
4. 项目实现了邮件、短信、支付、异步、图片一起的服务。

5. web端与移动共用一套接口

# 从数据库从查询某经纬度附近指定距离范围内数据
指定一个经纬度，给定一个范围值(单位:千米)，查出在经纬度周围这个范围内的数据。 

经度:113.914619

纬度:22.50128 

范围:2km 

longitude为数据表经度字段 

latitude为数据表纬度字段 

SQL在mysql下测试通过,其他数据库可能需要修改 

SQL语句如下: 

select * from location where sqrt( ( ((113.914619-longitude)*PI()*12656*cos(((22.50128+latitude)/2)*PI()/180)/180) * ((113.914619-longitude)*PI()*12656*cos (((22.50128+latitude)/2)*PI()/180)/180) ) + ( ((22.50128-latitude)*PI()*12656/180) * ((22.50128-latitude)*PI()*12656/180) ) )<2



MySQL性能调优 – 使用更为快速的算法进行距离

最近遇到了一个问题,通过不断的尝试最终将某句原本占据近1秒的查询优化到了0.01秒,效率提高了100倍.

问题是这样的,有一张存放用户居住地点经纬度信息的MySQL数据表,表结构可以简化 为:id(int),longitude(long),latitude()long. 而业务系统中有一个功能是查找离某个用户最近的其余数个用户,通过代码分析,可以确定原先的做法基本是这样的:

//需要查询的用户的坐标

$lat=20;

$lon=20;//执行查询,算出该用户与所有其他用户的距离,取出最近的10个

$sql='select * from users_location order by ACOS(SIN(('.$lat.' * 3.1415) / 180 ) *SIN((latitude * 3.1415) / 180 ) +COS(('.$lat.' * 3.1415) / 180 ) * COS((latitude * 3.1415) / 180 ) *COS(('.$lon.' * 3.1415) / 180 - (longitude * 3.1415) / 180 ) ) * 6380 asc limit 10';

而这条sql执行的速度却非常缓慢,用了近1秒的时间才返回结果,应该是因为order里的子语句用了太多的数学计算公式,导致整体的运算速度下降.

而在实际的使用中,不太可能会发生需要计算该用户与所有其他用户的距离,然后再排序的情况,当用户数量达到一个级别时,就可以在一个较小的范围里进行搜索,而非在所有用户中进行搜索.

所以对于这个例子,我增加了4个where条件,只对于经度和纬度大于或小于该用户1度(111公里)范围内的用户进行距离计算,同时对数据表中的经度和纬度两个列增加了索引来优化where语句执行时的速度.

最终的sql语句如下

$sql='select * from users_location where

latitude > '.$lat.'-1 and

latitude < '.$lat.'+1 and

longitude > '.$lon.'-1 and

longitude < '.$lon.'+1

order by ACOS(SIN(('.$lat.' * 3.1415) / 180 ) *SIN((latitude * 3.1415) / 180 ) +COS(('.$lat.' * 3.1415) / 180 ) * COS((latitude * 3.1415) / 180 ) *COS(('.$lon.'* 3.1415) / 180 - (longitude * 3.1415) / 180 ) ) * 6380 asc limit 10';

经过优化的sql大大提高了运行速度,在某些情况下甚至有100倍的提升.这种从业务角度出发,缩小sql查询范围的方法也可以适用在其他地方.
