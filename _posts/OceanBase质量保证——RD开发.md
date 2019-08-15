# OceanBase质量保证——RD开发

OceanBase系统一直在不断演化，需要在代码不断变化的过程中保持系统的稳定性。因此，合理的质量保证体系关乎系统的成败。为了保证系统质量，OceanBase做了大量工作，在RD（指开发工程师）开发、QA（指测试工程师）测试、上线试运行各个阶段对系统质量把关。

系统Bug暴露越早修复代价越低，开发工程师是产生Bug的源头，开发阶段主要通过编码规范、代码审核（Code Revicw）、单元测试保证代码质量。另外，系统提测前RD需要主动执行快速测试（quicktest），从而避免返工。
##编码规范
编码规范规定了函数、变量、类型的命名规则，保证统一的注释和排版风格。除此之外，为了避免C/C++服务器端编程常见缺陷，OceanBase编码规范还制定了一些规则，如下所示：

1）	一个函数只能有一个人口和一个出口。不允许在函数中使用goto语句，也不允许函数中途return返回。如下所示，上边的代码中途调用了return，在OceanBase编码规范中是不允许的，可以修改为下边的方式。这条规定有一定的争议，很多优秀的开源项目都允许函数中途return。之所以这么规定，是为了确保函数执行过程中申请的资源被释放掉。对于分布式存储系统，代码稳定运行的重要性远远高于代码写得更漂亮。
```C++
Alloc_memory()；
If（x > 0）
{
  do_something1()；
  free_memory()；
  return true；
}
else
{
  do something2()；
  free_memory()；
  return false；
}
```  
```C++
boolean ret = true;
alloc_memory()；
If（x > 0）
{
  do_something1()；
  ret = true;
}
else
{
  do something2()；
  ret = false;
}
free_memory()；
return ret；
```  
2）禁止在函数中抛异常，谨慎使用STL、boost。C/C++编程的麻烦之处在于资源管理，尤其是内存管理。STL、boost库接口容易使用，能够提高编码效率，但是内存管理混乱，不易调试，且大多数开发工程师不了解其内部实现，不适用于高性能服务器的开发。

3）资源管理做到可控。所有的内存申请操作都需要经过OccanBase全局内存管理器，不允许直接在代码中调用new/malloc申请内存。另外，系统初始化时启动所有线程，执行过程中不允许动态启动额外的线程。

4）每个可能失败的函数都必须返回错误码，0表示成功，其他值表示出错。调用者需要仔细、全面地处理调用函数返回的每个错误码。

5）所有的指针使用前都必须判空，不允许使用assert语句替代错误检查。这条规定是为了保证程序执行过程中出现异常情况时能够打印错误日志而不是core dump。

6）不允许使用strcpy/strca/strcpy/sprintf等字符串操作函数，而改用对应的限制字符申长度函数：strncpy/strncat/strncpy/snprintf，从而防止字符串操作越界。

7）严格要求自己，编译时要开启GCC所有报警开关，例如：-Wall-Werror
-Wextra-Wunused-paramcter-Wformat -Wconversion -Wdeprecated。代码提交前需要确保解决所有的报警。

## 代码审核
OceanBase开发时要求所有代码提交前至少由一人审核，对于关键代码改动，例如，紧急修复线上Bug，需要架构师和各个小组的技术负责人参与。

代码审核工作主要包含两个部分：编码风格审核，比如是否符合编码规范，接口设计是否合理，以及实现逻辑审核。其中，实现逻辑审核是难点，要求理解每个代码实现细节，并给出建设性意见。每个刚刚加入团队的新人都会分配一个师兄，师兄的其中一项职责就是审核新人的代码，与新人一起共同对代码质量负责。

OceanBase采用开源的ReviewBoard(http://www.reviewboard.org/) 作为代码审核系统。
## 单元测试
OceanBase 采用googletest以及google mock进行单元测试。单元测试的关键点在于系统接口设计时考虑可测性，并提高每个开发人员的单元测试意识。

OccanBase单元测试已接入一沟网内部开发的Toast平台，每天晚上会自动回归所有的单元测试用例。Toast平台说明文档见：http://testing.etao.com/book/export/html/285 。 

## 快速测试（quicktest）
快速测试选取所有测试用例的一个子集，这个子集中的每个用例执行都很快，从而做到快速回归。快速测试部署成定时任务，每天自动回归，RD提交某个功能的代码之前也会主动运行快速测试，从而使得主干代码保持基本稳定。
## RD压力测试
### 分布式存储引擎压力测试
分布式存储引擎压力测试工具包含两个：syschecker 以及mixed_test。在syschecker工具中，多个客户端并发读写一行或者多行数据，并对读取到的每行数据进行校验。对于每行数据，其中的每一列都对应一个辅助列，二者数据之和为0。假设某列数据出错，syschecker能够很快检测出来。

syschecker写人速度很快，能够发现分布式存储引擎中的大部分问题，然面，syshecker只校验单行数据，不校验多行数据之间的关系。因此，syschecker无法发现某行数据全部丢失的情况。mixed_test正是用来解决这个问题的，它不仅对每行数据进行校验，还校验多行数据之间的关系，能够检测出某行数据全部丢失的情况。当然，Mixed_test 写入速度较慢，syschecker 和mixed_test两个工具总是配合使用。各有优势。
### 数据库功能压力测试
数据库功能压力测试工具包含两个：sqltest以及bigquery。
- sgltest工具测试时将指定一些SQL语句，sqltest工具会将这些语句分别发送给MySQL以及OceanBase数据库。如果二者的执行结果相同，则认为sqltest测试通过；否则，测试失败。
- bigquery工具是sqltest工具的补充，专门用于测试OLAP并发查询功能。Bigquery中每个查询涉及的数据往往跨多个子表，能够触发OceanBase的并发查询功能。当然，bigquery灵活性不够，只能执行特定的SQL语句，而sqltest能够执行OceanBase支持的所有SOL语句。因此，bigquery 和sqltest两个工具也是配合使用，各有优势。
