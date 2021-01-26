for i in `docker ps -a |grep -i exited |awk '{print $1}'`;do docker rm -f $i;done
