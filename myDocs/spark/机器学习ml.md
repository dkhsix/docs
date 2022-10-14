### spark mllib 机器学习构成

#### 两个算法包

- spark.mllib:包含原始api,构建在RDD之上

- spark.ml:基于DataFrame构建的高级API

#### 如何选择

- spark.ml具备更忧的性能和更好的扩展性，建仪优先选用
- spark.mllib仍将继续更新，且目前包含更多（相比于spark.ml）的算法

### mllib构成

#### 数据类型（data type）

- 向量，带类别的向量，矩阵等

#### 数据统计计算库

- 基本统计量（min,max,average等），相关分析，随机数产生器，假设检验等

#### 机器学习管道(pipeline)

- Transformer (转换器)

- Estimator (评估器)

- Parameter (参数)

#### 机器学习算法

- 分类算法

- 回归算法

- 聚类算法

- 协同算法

#### 数据类型

##### Vector

- Dense vector (稠密向量)
- Sparse vector(稀疏向量)
- Labeled point(标记 点) 带标签的向量

##### Matrix

- Local Matrix 本地矩阵 单机的
- Distributed Matrix 分布式向量,矩阵