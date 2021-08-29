# RocksDB Festival : DKU StarLab



## BGR Info
#### What does BGR mean?
> Big data, Guru, RocksDB

#### Team Member (Github ID)
> 황예진(hyj3463), 박경미(kmi0817)  
> hyj3463@naver.com, kmi0817@naver.com
#### Team Research
> Write/Read/Space Amplification




## BGR's Goal
- We look for factors that affect write/read/space amplification in Rocksdb and discuss ways to reduce that amplification.
- In addition, we understand the internal structure of Rocksdb.




## Experimental and Study Objectives
- #### Weeks 1 to 2 (July 5 to July 12)  
> Rocksdb Lecture  
- #### Weeks 3-4 (July 19 to July 26)  
> Understanding db_bench  
- #### Weeks 5–6 (August 2 – August 9)  
> SSTable & Memtable  
- #### Week 7  
> Vacation  
- #### Weeks 8–9 (August 23 – August 30)  
> level_compaction_dynamic_level_bytes option  




## Weeks 3-4 : Use db_bench
- db_bench 명령어 공부
- 실험
1. key Size 변경 (6B/64B/128B/256B/1KB/4KB)  
![image](https://user-images.githubusercontent.com/58660316/130969108-c07edc1e-7e5d-44b1-922f-93100471bbed.png)![image](https://user-images.githubusercontent.com/58660316/130969115-4b2cfb1c-8bb3-4f55-b6c8-a4303e3c9c37.png)
- Key Size가 커질 수록 처리율이 높아진다. 특히 245B에서 1KB으로 넘어갈 때 급격히 높아진다.
(1) RocksDB에서 sequential이 random보다 빠르고, 쓰기 속도(write)가 읽기 속도(read)보다 빠르다. 



## Weeks 5-6 : Write Amplification
![image](https://user-images.githubusercontent.com/58660316/130989774-ac89e27e-bd35-40bb-8bc1-49d066fbe952.png)  
Write Amplification을 줄이고 성능을 증가시키는 요인으로 Memtable과 SSTable의 크기를 선택해 실험했다.  
### 1. Memtable Size 변경 (64MB/128MB/256MB)  
![130989832-2e2c1d1b-cfa7-4ed3-9790-b7a657cc9032](https://user-images.githubusercontent.com/58660316/130991308-688f21e5-814a-4194-9f89-194932ff80cc.png)   
Memtable 크기를 크게하면 Disk의 L0로 Compaction되는 횟수가 적어질거라 예상했고, 동시에 Write Amplification이 줄어들 것이라 예상했다.
```
./db_bench --benchmarks="fillrandom,levelstats,stats" --key_size=16 --value_size=1024 --num=1000000~5000000 --write_buffer_size=67108864~268435456
```
![image](https://user-images.githubusercontent.com/58660316/130990220-f9f0a9ce-c3d2-4497-a8d0-ebb80032c6b4.png)  
결과는 Memtable 크기가 커질수록 random write에서는 성능이 증가했으나 반대로 sequential write에서는 성능이 감소했다.  
또한 Write Amplification이 sequential write는 이론상 1, random write에서는 감소했고, Copmaction 또한 대폭 감소했다.

### 2. SSTable Size 변경 (64MB/16MB/4MB)  
![image](https://user-images.githubusercontent.com/58660316/130989869-06591dee-5408-47f0-a54e-9ee854a840bf.png)  
SSTable 크기를 작게하면 각 레벨에 있는 SST file의 개수가 늘어나고 각 SST file이 가지고 있는 Key range가 줄어들 것이다. 따라서 Compaction 시 합병할 SST file을 선택하는데 있어 오차 범위를 줄일 수 있고, Write Amplification 또한 줄어들 것이라 예상했다.  
```
./db_bench --benchmarks="fillrandom,levelstats,stats" --key_size=16 --value_size=1024 --num=5000000 --target_file_size_base=1048576~67108864
```
![image](https://user-images.githubusercontent.com/58660316/130990416-7057950a-7c51-4453-9427-d92cc035f80b.png)  
![image](https://user-images.githubusercontent.com/58660316/130990421-7c53be06-1a83-4e39-90b0-c4db32eb35f7.png)  
결과는 SSTable 크기가 작을수록 random write 또한 성능이 증가했다. 그러나 sequential write에서는 오히려 크기가 큰 SSTable에서 성능이 높았는데, 이는 Workload에 따라 차이가 있는 것으로 보인다.
또한 SSTable 크기가 작을수록 Compaction 횟수나 Write Amplification도 줄었다. 이때 16MB 아래로는 변화가 미미했다.  

### 3. Memtable & SSTable Size 변경  
```
./db_bench --benchmarks="fillrandom,levelstats,stats" --key_size=16 --value_size=1024 --num=5000000 --write_buffer_size=67108864~268435456 --target_file_size_base=1048576~67108864
```
Memtable과 SSTable의 크기를 동시에 변경했을 때 Memtable은 클수록, SSTable은 작을수록 성능이 증가했으며, Compaction이나 Write Amplification은 감소했다. 

발표 문서 : RocksDB_Festival_BGR_WA.pdf




## Weeks 8-9 : Space Amplification
Space Amplification을 줄이고 성능을 증가시키는 요인으로 level_compaction_dynamic_level_bytes 옵션과 max_bytes_for_level_multiplier 옵션을 선택해 실험했다.
사용한 Space Amplification 계산 공식은 Size of the entire level / Size of the last level 이다.

### 1. level_compaction_dynamic_level_bytes
#### level_compaction_dynamic_level_bytes is FALSE  
![다이나믹 false](https://user-images.githubusercontent.com/58660316/130994069-dfdedc5f-b1be-40c2-9479-8e0e9de44691.JPG)  
옵션을 활성화하지 않으면 아래 레벨부터 데이터가 차례대로 적재된다.  
![false일 때 그림](https://user-images.githubusercontent.com/58660316/130992396-dbced256-499a-46da-8947-bf4d3cd21039.JPG)  
이때 각 레벨의 target size는 이전 레벨의 size * max_bytes_for_level_multiplier이다.


### 2. max_bytes_for_level_multiplier  
#### level_compaction_dynamic_level_bytes is TRUE  
```
./db_bench --benchmarks="fillrandom,levelstats,stats" --key_size=64 --value_size=1024 --num=5000000 --compression_type=none --compression_ratio=1 --level_compaction_dynamic_level_bytes=true --max_bytes_for_level_multiplier=2~10
```
![다이나믹 true](https://user-images.githubusercontent.com/58660316/130991880-7f1d3a3d-d996-4130-bcd7-d98223ddf8c8.JPG)  
옵션을 활성화했을 땐 max_bytes_for_level_base / max_bytes_for_level_multiplier의 값보다 작은 레벨은 데이터를 적재하지 않는다. L0의 Compaction 시에도 유효한 크기를 가진 레벨을 대상으로 진행한다.  
![true일 때 그림](https://user-images.githubusercontent.com/58660316/130992872-a857e61f-1f9e-4ce7-a07d-251fdf2e54c8.JPG)  
이때 각 레벨의 target size는 다음 레벨의 size / max_bytes_for_level_multiplier이다.  
그러나 이 경우 last Level의 target size를 정확히 알 수 없어 SA를 계산하지 못했다.  
![true일 때 disk](https://user-images.githubusercontent.com/58660316/130993707-0aa09b9a-94b8-454f-bd3c-051b677fb0b6.JPG)  
또한 https://github.com/facebook/rocksdb/wiki/Leveled-Compaction 의 결과처럼 last Level에 데이터의 90%가 쌓이지 않고 L5에 가장 많은 데이터가 쌓인 이유를 알아보는 중이다.


발표 문서 : RocksDB_Festival_BGR_SA.pdf







## Team BGR : Experiment Environment

#### 1. Yejin Hwang's Experiment Environment
```
OS(Operatring System) : Ubuntu 18.04
CPU : 1 * Intel(R) Core(TM) i5-7500 CPU @ 3.40GHz
RAM : 8 GB
SSD : ADATA SU800
RocksDB version 6.21
```

#### 2. Kyeongmi Park's Experiment Environment
```
OS(Operatring System) : Ubuntu 20.04.2 LTS
CPU : Intel(R) Core(TM) i5-8265U CPU @ 1.60GHz x 4
RAM : 8.00 GB
SSD : SN720 SDAQNTW0256G-1004
RocksDB ver. version 6.22
```
