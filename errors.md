When reconstrcting the cluster and reusing the same persistent storage I found lots of problems. I spend a lot of time restarting the same steps without changing nothing due to the fact that **I did not look to the logs**
The errors where that the persistant storage has images and data and sometimes restarting and changing thigs has provoques those errores of the pod (exit error number 1). So normally it has to do with app errors of the container

I don't know why but it seems to me that when we delete a disk in GCE and rebuild it with the same name it seems to recreats it with the same data?
