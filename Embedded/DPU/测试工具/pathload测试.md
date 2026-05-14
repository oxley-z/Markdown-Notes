# pathload测试

## 下载

[Pathload (gatech.edu)](https://www.cc.gatech.edu/fac/Constantinos.Dovrolis/bw-est/pathload.html)

## 编译

```bash
eddy@eddy  ~/test  tar -zxvf pathload.tar.gz 
pathload_1.3.2/
pathload_1.3.2/CHANGELOG
pathload_1.3.2/CHANGES
pathload_1.3.2/COPYING
pathload_1.3.2/README
pathload_1.3.2/config.guess
pathload_1.3.2/config.sub
pathload_1.3.2/configure
pathload_1.3.2/makefile.in
pathload_1.3.2/pathload_gbls.h
pathload_1.3.2/pathload_rcv.c
pathload_1.3.2/pathload_rcv.h
pathload_1.3.2/pathload_rcv_func.c
pathload_1.3.2/pathload_snd.c
pathload_1.3.2/pathload_snd.h
pathload_1.3.2/pathload_snd_func.c
eddy@eddy  ~/test  cd pathload_1.3.2 
eddy@eddy  ~/test/pathload_1.3.2  ./configure 
checking for gcc... gcc
checking whether the C compiler (gcc  ) works... yes
checking whether the C compiler (gcc  ) is a cross-compiler... no
checking whether we are using GNU C... yes
checking whether gcc accepts -g... yes
checking host system type... x86_64-unknown-linux-gnu
checking for the pthreads library -lpthreads... no
checking whether pthreads work without any flags... no
checking whether pthreads work with -Kthread... no
checking whether pthreads work with -kthread... no
checking for the pthreads library -llthread... no
checking whether pthreads work with -pthread... yes
checking for joinable pthread attribute... PTHREAD_CREATE_JOINABLE
checking if more special flags are required for pthreads... no
checking for cc_r... gcc
checking for main in -lm... yes
checking for main in -lposix4... no
checking for socket in -lsocket... no
checking for gethostbyname in -lnsl... yes
checking how to run the C preprocessor... gcc -E
checking for ANSI C header files... yes
checking for fcntl.h... yes
checking for strings.h... yes
checking for sys/time.h... yes
checking for unistd.h... yes
checking whether time.h and sys/time.h may both be included... yes
checking whether struct tm is in sys/time.h or time.h... time.h
checking size of int... 4
checking size of long... 8
checking for strftime... yes
checking for gethostname... yes
checking for gettimeofday... yes
checking for socket... yes
creating ./config.status
creating makefile
eddy@eddy  ~/test/pathload_1.3.2  make                  
gcc -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB   -c -o pathload_snd.o pathload_snd.c
pathload_snd.c: In function ‘main’:
pathload_snd.c:204:49: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  204 |       printf("Maximum packet size          :: %ld bytes\n",max_pkt_sz);
      |                                               ~~^          ~~~~~~~~~~
      |                                                 |          |
      |                                                 long int   l_int32 {aka int}
      |                                               %d
gcc -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB   -c -o pathload_snd_func.o pathload_snd_func.c
pathload_snd_func.c: In function ‘send_fleet’:
pathload_snd_func.c:62:53: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
   62 |     printf("ERROR : send_fleet : unable to malloc %ld bytes \n",cur_pkt_sz);
      |                                                   ~~^           ~~~~~~~~~~
      |                                                     |           |
      |                                                     long int    l_int32 {aka int}
      |                                                   %d
pathload_snd_func.c:69:29: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
   69 |     printf("Sending fleet %ld ",fleet_id);
      |                           ~~^   ~~~~~~~~
      |                             |   |
      |                             |   l_int32 {aka int}
      |                             long int
      |                           %d
pathload_snd_func.c: In function ‘send_latency’:
pathload_snd_func.c:285:55: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  285 |     printf("ERROR : send_latency : unable to malloc %ld bytes \n",max_pkt_sz);
      |                                                     ~~^           ~~~~~~~~~~
      |                                                       |           |
      |                                                       long int    l_int32 {aka int}
      |                                                     %d
pathload_snd_func.c: In function ‘send_train’:
pathload_snd_func.c:345:53: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  345 |     printf("ERROR : send_train : unable to malloc %ld bytes \n",max_pkt_sz);
      |                                                   ~~^           ~~~~~~~~~~
      |                                                     |           |
      |                                                     long int    l_int32 {aka int}
      |                                                   %d
gcc pathload_snd.o pathload_snd_func.o -o pathload_snd -lnsl -lm    -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB
gcc -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB   -c -o pathload_rcv.o pathload_rcv.c
pathload_rcv.c: In function ‘main’:
pathload_rcv.c:275:49: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  275 |     printf("  Maximum packet size          :: %ld bytes\n",max_pkt_sz);
      |                                               ~~^          ~~~~~~~~~~
      |                                                 |          |
      |                                                 long int   l_int32 {aka int}
      |                                               %d
pathload_rcv.c:276:60: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  276 |   fprintf(pathload_fp,"  Maximum packet size          :: %ld bytes\n",max_pkt_sz);
      |                                                          ~~^          ~~~~~~~~~~
      |                                                            |          |
      |                                                            long int   l_int32 {aka int}
      |                                                          %d
pathload_rcv.c:281:49: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  281 |     printf("  send latency @sndr           :: %ld usec\n",snd_latency);
      |                                               ~~^         ~~~~~~~~~~~
      |                                                 |         |
      |                                                 long int  l_uint32 {aka unsigned int}
      |                                               %d
pathload_rcv.c:282:49: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  282 |     printf("  recv latency @rcvr           :: %ld usec\n",rcv_latency);
      |                                               ~~^         ~~~~~~~~~~~
      |                                                 |         |
      |                                                 long int  l_uint32 {aka unsigned int}
      |                                               %d
pathload_rcv.c:284:60: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  284 |   fprintf(pathload_fp,"  send latency @sndr           :: %ld usec\n",snd_latency);
      |                                                          ~~^         ~~~~~~~~~~~
      |                                                            |         |
      |                                                            long int  l_uint32 {aka unsigned int}
      |                                                          %d
pathload_rcv.c:285:60: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  285 |   fprintf(pathload_fp,"  recv latency @rcvr           :: %ld usec\n",rcv_latency);
      |                                                          ~~^         ~~~~~~~~~~~
      |                                                            |         |
      |                                                            long int  l_uint32 {aka unsigned int}
      |                                                          %d
pathload_rcv.c:291:49: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  291 |     printf("  Minimum packet spacing       :: %ld usec\n",min_time_interval );
      |                                               ~~^         ~~~~~~~~~~~~~~~~~
      |                                                 |         |
      |                                                 long int  l_uint32 {aka unsigned int}
      |                                               %d
pathload_rcv.c:292:60: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_uint32’ {aka ‘unsigned int’} [-Wformat=]
  292 |   fprintf(pathload_fp,"  Minimum packet spacing       :: %ld usec\n",min_time_interval );
      |                                                          ~~^         ~~~~~~~~~~~~~~~~~
      |                                                            |         |
      |                                                            long int  l_uint32 {aka unsigned int}
      |                                                          %d
gcc -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB   -c -o pathload_rcv_func.o pathload_rcv_func.c
pathload_rcv_func.c: In function ‘recvfrom_latency’:
pathload_rcv_func.c:45:40: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
   45 |     printf("ERROR : unable to malloc %ld bytes \n",max_pkt_sz);
      |                                      ~~^           ~~~~~~~~~~
      |                                        |           |
      |                                        long int    l_int32 {aka int}
      |                                      %d
pathload_rcv_func.c: In function ‘recv_train’:
pathload_rcv_func.c:197:40: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  197 |     printf("ERROR : unable to malloc %ld bytes \n",max_pkt_sz);
      |                                      ~~^           ~~~~~~~~~~
      |                                        |           |
      |                                        long int    l_int32 {aka int}
      |                                      %d
pathload_rcv_func.c: In function ‘recv_fleet’:
pathload_rcv_func.c:366:40: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  366 |     printf("ERROR : unable to malloc %ld bytes \n",cur_pkt_sz);
      |                                      ~~^           ~~~~~~~~~~
      |                                        |           |
      |                                        long int    l_int32 {aka int}
      |                                      %d
pathload_rcv_func.c:372:31: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  372 |     printf("Receiving Fleet %ld, Rate %.2fMbps\n",exp_fleet_id,tr);
      |                             ~~^                   ~~~~~~~~~~~~
      |                               |                   |
      |                               long int            l_int32 {aka int}
      |                             %d
pathload_rcv_func.c:375:33: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 2 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  375 |     printf("\nReceiving Fleet %ld\n",exp_fleet_id);
      |                               ~~^    ~~~~~~~~~~~~
      |                                 |    |
      |                                 |    l_int32 {aka int}
      |                                 long int
      |                               %d
pathload_rcv_func.c:376:12: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  376 |     printf("  Fleet Parameter(req)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, \
      |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  377 | T=%ldusec\n",tr, cur_pkt_sz , stream_len,time_interval) ;
      | ~~~~~~~~~~~~     ~~~~~~~~~~
      |                  |
      |                  l_int32 {aka int}
pathload_rcv_func.c:376:12: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 4 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
pathload_rcv_func.c:376:12: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 5 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
pathload_rcv_func.c:379:44: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  379 |   fprintf(pathload_fp,"\nReceiving Fleet %ld\n",exp_fleet_id);
      |                                          ~~^    ~~~~~~~~~~~~
      |                                            |    |
      |                                            |    l_int32 {aka int}
      |                                            long int
      |                                          %d
pathload_rcv_func.c:380:23: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 4 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
  380 |   fprintf(pathload_fp,"  Fleet Parameter(req)  :: R=%.2fMbps, L=%ldB, \
      |                       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  381 | K=%ldpackets, T=%ldusec\n",tr, cur_pkt_sz , stream_len,time_interval);
      | ~~~~~~~~~~~~~~~~~~~~~~~~~~     ~~~~~~~~~~
      |                                |
      |                                l_int32 {aka int}
pathload_rcv_func.c:380:23: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 5 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
pathload_rcv_func.c:380:23: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 6 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
pathload_rcv_func.c: In function ‘get_sending_rate’:
pathload_rcv_func.c:1602:56: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 3 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1602 |     printf("  Fleet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate, cur_pkt_sz , stream_len,time_interval);
      |                                                      ~~^                                               ~~~~~~~~~~
      |                                                        |                                               |
      |                                                        long int                                        l_int32 {aka int}
      |                                                      %d
pathload_rcv_func.c:1602:64: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 4 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1602 |   printf("  Fleet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate, cur_pkt_sz , stream_len,time_interval);
      |                                                            ~~^                                                    ~~~~~~~~~~
      |                                                              |                                                    |
      |                                                              long int                                             l_int32 {aka int}
      |                                                            %d
pathload_rcv_func.c:1602:78: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 5 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1602 | eet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate, cur_pkt_sz , stream_len,time_interval);
      |                                                            ~~^                                                 ~~~~~~~~~~~~~
      |                                                              |                                                 |
      |                                                              long int                                          l_int32 {aka int}
      |                                                            %d
pathload_rcv_func.c:1603:67: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 4 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1603 | intf(pathload_fp,"  Fleet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate,cur_pkt_sz,stream_len,time_interval);
      |                                                            ~~^                                              ~~~~~~~~~~
      |                                                              |                                              |
      |                                                              long int                                       l_int32 {aka int}
      |                                                            %d
pathload_rcv_func.c:1603:75: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 5 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1603 | hload_fp,"  Fleet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate,cur_pkt_sz,stream_len,time_interval);
      |                                                            ~~^                                                 ~~~~~~~~~~
      |                                                              |                                                 |
      |                                                              long int                                          l_int32 {aka int}
      |                                                            %d
pathload_rcv_func.c:1603:89: warning: format ‘%ld’ expects argument of type ‘long int’, but argument 6 has type ‘l_int32’ {aka ‘int’} [-Wformat=]
 1603 | eet Parameter(act)  :: R=%.2fMbps, L=%ldB, K=%ldpackets, T=%ldusec\n",cur_actual_rate,cur_pkt_sz,stream_len,time_interval);
      |                                                            ~~^                                              ~~~~~~~~~~~~~
      |                                                              |                                              |
      |                                                              long int                                       l_int32 {aka int}
      |                                                            %d
gcc pathload_rcv.o pathload_rcv_func.o  -o pathload_rcv -lnsl -lm    -g -O2 -pthread  -DHAVE_PTHREAD=1 -DHAVE_LIBM=1 -DHAVE_LIBNSL=1 -DSTDC_HEADERS=1 -DHAVE_FCNTL_H=1 -DHAVE_STRINGS_H=1 -DHAVE_SYS_TIME_H=1 -DHAVE_UNISTD_H=1 -DTIME_WITH_SYS_TIME=1 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_STRFTIME=1 -DHAVE_GETHOSTNAME=1 -DHAVE_GETTIMEOFDAY=1 -DHAVE_SOCKET=1  -DTHRLIB
rm -f pathload_snd.o pathload_snd_func.o pathload_rcv.o pathload_rcv_func.o  config.cache config.log  config.status

```



server

```bash
./pathload_snd
```

client

```bash
./pathload_rcv -s <server ip>
```

