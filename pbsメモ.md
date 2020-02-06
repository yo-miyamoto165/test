#### インストール前提条件

- ずべてのPBSデーモンが同一バージョン

- ホスト間の名前解決

- パスワード不要の scp or rcp の設定 (標準出力/標準エラーのデリバリー準備)

- システム時刻の統一

- rootアカウントでインストール

- (商用のみ) DB用アカウント準備

  ユーザー名(デフォルト) : pbsdata ※UID 1000以下



#### コマンド

- ノードの状態

  - qstat -Bf

    ```
    # qstat -Bf
    Server: hpc
        server_state = Idle
        server_host = hpc
        total_jobs = 0
        state_count = Transit:0 Queued:0 Held:0 Waiting:0 Running:0 Exiting:0 Begun
            :0
        default_chunk.ncpus = 1
        scheduler_iteration = 600
        FLicenses = 20000000
        resv_enable = True
        node_fail_requeue = 310
        max_array_size = 10000
        pbs_license_min = 0
        pbs_license_max = 2147483647
        pbs_license_linger_time = 31536000
        license_count = Avail_Global:10000000 Avail_Local:10000000 Used:0 High_Use:
            0
        pbs_version = 19.1.2
        eligible_time_enable = False
        max_concurrent_provision = 5
        power_provisioning = False
        max_job_sequence_id = 9999999
    ```

  - pbsnodes -av

  - pbsnodes <node名>

    ```
    # pbsnodes hpc01
    hpc01
         Mom = hpc01
         Port = 15002
         pbs_version = 19.1.2
         ntype = PBS
         state = free
         pcpus = 1
         resources_available.arch = linux
         resources_available.host = localhost
         resources_available.mem = 1014276kb
         resources_available.ncpus = 1
         resources_available.Qlist = HPC
         resources_available.vnode = hpc01
         resources_assigned.accelerator_memory = 0kb
         resources_assigned.hbmem = 0kb
         resources_assigned.mem = 0kb
         resources_assigned.naccelerators = 0
         resources_assigned.ncpus = 0
         resources_assigned.vmem = 0kb
         resv_enable = True
         sharing = default_shared
         last_state_change_time = Wed Feb  5 15:24:43 2020
    ```

    > ###### 見方
    >
    > resources_available.ncpus = 1		# hpc01で使用可能なCPU数
    > resources_assingned.ncpus = 0 	# ジョブに割り当てられたCPU数
    >
    > →ジョブの割当可否：available - assinged



- ジョブの状態

  - ジョブ一覧

    ```
    # qstat
    ```

    > ###### オプション
    >
    > -x 	# ジョブヒストリーを有効にしている場合に実行終了後のジョブを含めて照会
    > -a     # ジョブ名、セッションIDなど詳細表示
    > -s      # -a同等にコメントを付加
    > -n      # -a同等にジョブ実行中のvnode付加
    >
    > >###### ジョブのステータス一覧
    > >
    > >Q : キュー待ち
    > >R : 実行中
    > >S : サスペンド
    > >E : 実行後、Exit状態
    > >H : ホールド
    > >F : 終了 (成功したかは無関係)
    > >M: 他のPBSコンプレックスに移動
    > >W: リクエストされた実行時刻まで待機中
    > >      または実行前のプロセスにおけるエラー

  - ジョブ詳細

    ```
    # qstat -f 26
    Job Id: 26.hpc01
     Job_Name = STDN
     <中略>
     Resource_List.ncpus = 1
     <以下略>
    ```

    > ###### 見方
    >
    > Resource_available.ncpus = 1	# ジョブに割り当てられたcpu数

    ```
    # tracejob -n<days> <jobid>
    ```

    ex) tracejob -n4 6.pbsserver

  

  

- ジョブ終了

  - qdel <jobid>
  - qdel -W force <jobid>   # Eステータスのジョブを強制削除



### リソースの概念

- リソース状態の確認

  ノードで使用可能なリソース、割り当て状況

  ```
  # pbsnodes -a
  hpc03
  	Mom = hpc03
  	
  	resources_available.mem = 772012kb  
  	resources_available.ncpus = 4			# hpc03で使用可能なCPU数
  	resources_available.vnode = hpc03
  	resources_assigned.mem = 262144kb
  	resources_available.ncpus = 2 			# hpc03で現在使用されているCPU数
  ```

  ジョブが使用するリソース

  ```
  # qstat -f 26
  Job Id: 26.hpc03
  Job_Name =
  Job_Owner = hpc@hpc
  
  Resource_List.mem = 262144kb
  Resource_List.ncps = 2
  ```

- chunk と select

  ```
  select=4:ncpus=2			# 8並列のMPIジョブ (4node * 2CPU)
  select=1:ncpus=2+3:ncpus=1	# 4並列のMPIジョブ (chunkの内容が異なる場合は+を使用)
  ```

- place

  |          | 属性      | 意味                                              |
  | -------- | --------- | ------------------------------------------------- |
  | type     | free      | chunkはあらゆるvnodeに配置可能 (デフォルト)       |
  |          | pack      | すべてのchunkを1ホスト上に配置                    |
  |          | scatter   | 1ホスト上には1chunkのみ                           |
  | sharing  | exclusive | vnodeを排他的に利用                               |
  |          | shared    | 他のジョブとvnodeを共用                           |
  | grouping | group=res | chunkを指定したリソースのグループ分けに準じて配置 |

  ```
  							# node1 node2 node3 node4 node5 node6 node7 node8
  # default(=free)
  -l select=4:ncpus=1			#  12    34    □□    □□    □□    □□    □□     □□
  
  # free
  -l select=4:ncpus=1
    -l place=free				#  12    34    □□    □□    □□    □□    □□     □□
    
  # scatter
  -l select=4:ncpus=1
    -l place=scatter			#  1□	 2□	   3□    4□    □□    □□    □□     □□
    #scatter2ジョブ目
  -l select=4:ncpus=1
    -l place=scatter			#  ■5	 ■6	   ■7    ■8    □□    □□    □□     □□
    
  # pack
  -l select=2:ncpus=1
    -l place=pack				#  ■□	 ■□	   ■□    ■□    12    □□    □□     □□
    
  # scater:excl
  -l select=4:ncpus=1
    -l place=scatter:excl		#  ■□	 ■□	   ■□    ■□    1x    2x    3x     4x
  # scater:shared
  -l select=4:ncpus=1
    -l place=scatter:shared	#  ■□	 ■□	   ■□    ■□    1□    2□    3□     4□
  
  # chunkは分割されない
  -l select=2:ncpus=2			#  ■□	 ■□	   ■□    ■□    11    22    □□     □□
  	-l place=free			※以下にはならない
                              #  ■1	 ■1	   ■2    ■2    □□    □□    □□     □□
  ```





### MPIジョブスクリプトサンプル

```
#!/bin/bash
#PBS -l nodes=4:ppn=12

cd $PBS_O_WORKDIR
uniq -c $PBS_NODEFILE | awk '{ print($2,"slots="$1)}' > hostsfile
mpirun -machinefile hostsfile -np 48 ./a.out
```

```
#!/bin/bash
#PBS -l nodes=2:ppn=1

cd $PBS_O_WORKDIR
echo $PBS_NODEFILE > hostsfile
uniq -c $PBS_NODEFILE >> hostsfile
uniq -c $PBS_NODEFILE | awk '{ print($2,"slots="$1)}' >> hostsfile
```



#### PBS変数

Variable	Description

--------------- -----------------------------------------------------------------------------------------------------------
| Variable      | Description                                                  | ex)         |
| ------------- | ------------------------------------------------------------ | ----------- |
| PBS_JOBNAME   | User specified jobname                                       | pbs_loop.sh |
| PBS_O_WORKDIR | Job s submission directory                                   | /home/hpc   |
| PBS_TASKNUM   | Number of tasks requested                                    | 1           |
| PBS_O_HOME    | Home directory of submitting user                            | /home/hpc   |
| PBS_MOMPORT   | Active port for mom daemon                                   | 15003       |
| PBS_O_LOGNAME | name of submitting user                                      | hpc         |
| PBS_NODENUM   | Node offset number                                           | 0           |
| PBS_O_SHELL   | Script shell                                                 | /bin/bash   |
| PBS_O_JOBID   | Unique pbs job id                                            |             |
| PBS_O_HOST    | Host on which job script is currently running                | hpc01       |
| PBS_QUEUE     | Job queue                                                    | workq       |
| PBS_NODEFILE  | File contain in g li ne delim i ted list on nodes allocated to the job | *1          |
| PBS_O_PATH    | Path variable used to locate executables within job script   | *2          |

*1 > $ cat /var/spool/pbs/aux/10.hpc01
           hpc01
           hpc01
	       hpc01
           hpc01
	       hpc02
           hpc02
	       hpc02
	       hpc02

*2 > /usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/opt/pbs/bin:/home/hpc/.local/bin:/home/hpc/bin