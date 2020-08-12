第一个问题：如何check ubuntu16.04 有没有安装boost，如果有安装，那么boost是什么版本？


第一种方法：dpkg -s libboost-dev | grep Version

![boost version1](imgs/boost/boost_version_method1.png "boost version1")

第二种方法：cat /usr/include/boost/version.hpp | grep "BOOST_LIB_VERSION"

![boost version2](imgs/boost/boost_version_method2.png "boost version2")
