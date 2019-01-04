---
layout: post
title: InfoSec:恶意URL检测
category: InfoSec_LAB
description: InfoSec:恶意URL检测
published: true
---

## 实验简介
通过向服务器发送恶意URL，实现攻击是一种常见的攻击手段，本次实验就要通过使用机器学习的方法来检测恶意URL


## 实验平台和工具
Windows 10
Python 3

## 实验步骤
### 使用的python依赖库
>numpy
scikit-learn
scipy
sklearn

### 数据
good_dataset: 100000 个正常URL
bad_dataset: 40000 个恶意URL
训练集：60% 
测试集：40%

正常URL:

![image.png-178.5kB][1]

恶意URL:

![image.png-339.5kB][2]

### 1.文本向量化
要使用机器学习的方法进行分类，我们必须将原始的文本向量化，从而可以作为分类的特征进行使用

**人工筛选特征**
为每个URL构造一个四维特征：

- URL长度
- 是否包含（http://)或(https://）
- 恶意字符个数
- 恶意单词个数

```
def get_feature(datas):
    url_len = []
    url_count = []
    evil_char = []
    evil_word = []
    for data in datas:
        url_len.append(len(data))
        if re.search('(http://)|(https://)', data, re.IGNORECASE):
            url_count.append(1)
        else:
            url_count.append(0)
        evil_char.append(len(re.findall("[<>,\'\"/]", data, re.IGNORECASE)) +\
                         len(re.findall("[-*]", data, re.IGNORECASE)) +\
                         len(re.findall("[${=}.|]", data, re.IGNORECASE)))
        evil_word.append(len(re.findall("(alert)|(script=)(%3c)|(%3e)|(%20)|(onerror)|(onload)|(eval)|(src=)|(prompt)|(ob_start)|(shell_exec)|(passthru)|(escapeshellcmd)|(proc_popen)|(pcntl_exec)|(phpinfo)|(exit)", data, re.IGNORECASE)) +\
                         len(re.findall("(print)|(assert)|(system)|(preg_replace)|(create_function)|(call_user_func)|(call_user_func_array)|(eval)|\
                         (array_map)", data, re.IGNORECASE)) + len(re.findall("(chr)|(%27)|(passwd)|(whoami)|(base64_decode)", data, re.IGNORECASE)))
    feature = np.reshape(url_len + url_count + evil_char + evil_word, (4, len(url_len))).transpose()#四个特征
    return feature
```

**TF-IDF特征**
TF-IDF即词频-逆文本频率，TF表示词频，指的是某一个给定的词语在该文件中出现的频率。DF逆向文件频率（inverse document frequency，idf）是一个词语普遍重要性的度量。
tf-idf特征倾向于过滤掉常见的词语，保留重要的词语。

```
def get_tfidf(datas):
    tfidf_model = TfidfVectorizer(min_df=0.0, analyzer="char", sublinear_tf=True,
                                  ngram_range=(1, 1)).fit(datas)
    datas = tfidf_model.transform(datas)
    return datas
```

### 2.使用SVM和逻辑回归进行分类
**SVM**
```
def svm_model(datas, labels):
    start_time = time.time()  # 获取开始时间
    model_name = "model_exefile.pkl"
    # 总共240000条数据，100000条正常数据，40000条异常数据，生成训练数据集和测试数据集，训练集60%，测试集40%
    x_train, x_test, y_train, y_test = model_selection.train_test_split(datas, labels, test_size=0.4, random_state=0)
    clf = svm.SVC(kernel='rbf', verbose=True, shrinking=False).fit(x_train, y_train)
    joblib.dump(clf, model_name)
    y_pred = clf.predict(x_test)
    show_result(y_test, y_pred)
    end_time = time.time()
    print("训练时间为：" + str(end_time - start_time))
```

**Logistic Regression**
```
def LR_model(datas,labels):
    model_name = "model_lr.pkl"
    x_train, x_test, y_train, y_test = model_selection.train_test_split(datas, labels, test_size=0.4, random_state=0)
    lgs = LogisticRegression(class_weight={1: 2 * 40000 / 100000, 0: 1.0}).fit(x_train, y_train)
    joblib.dump(lgs, model_name)
    result = lgs.predict(x_test)
    show_result(y_test, result)
```

**打印结果**
```
def show_result(y_test,y_pred):
    print("测试数据准确率:")
    print(metrics.accuracy_score(y_test, y_pred))
    print("结果矩阵:")
    print(metrics.confusion_matrix(y_test, y_pred))
    print("正确率:")
    print(metrics.precision_score(y_test, y_pred))
    print("召回率:")
    print(metrics.recall_score(y_test, y_pred))
    print("F1值:")
    print(metrics.f1_score(y_test, y_pred))
```

### 3.测试两种分类器
```
#载入数据
bad_dataset, bad_labels = load_data("data/badqueries.txt", 0)
good_dataset, good_labels = load_data("data/goodqueries.txt", 1)
datas = bad_dataset[:40000] + good_dataset[:100000]
labels = np.concatenate((bad_labels[:40000], good_labels[:100000]), axis=0)
```

```
tf_idf = get_tfidf(datas).toarray()#tfidf特征
svm_model(tf_idf, labels)
```
>[LibSVM]测试数据准确率:
0.9440535714285714
结果矩阵:
[[13618  2291]
 [  842 39249]]
正确率:
0.9448483389504092
召回率:
0.9789977800503854
F1值:
0.9616199728044492
训练时间为：470.1522378921509

```
LR_model(tf_idf, labels)
```
>测试数据准确率:
0.9561428571428572
结果矩阵:
[[14324  1585]
 [  871 39220]]
正确率:
0.9611567209900748
召回率:
0.9782744256815744
F1值:
0.9696400316455697

```
feature = get_feature(datas)#人工定义特征
svm_model(feature, labels)
```
>[LibSVM]测试数据准确率:
0.9375357142857143
结果矩阵:
[[13330  2579]
 [  919 39172]]
正确率:
0.9382290244545041
召回率:
0.9770771494849217
F1值:
0.9572591090149312
训练时间为：117.88093996047974

```
LR_model(feature, labels)
```
>测试数据准确率:
0.921
结果矩阵:
[[12463  3446]
 [  978 39113]]
正确率:
0.9190300523978477
召回率:
0.9756054974932029
F1值:
0.9464730792498488


  [1]: http://static.zybuluo.com/windmelon/pbsxxpittx0sj8w88qw97rcf/image.png
  [2]: http://static.zybuluo.com/windmelon/ehc8z998qf1f07bvow8ra2kd/image.png
