# Core dump

The core dump feature allows the operating system to store the current memory status of a program to a file when the program terminates abnormally.

## Background information

The Linux operating system generates a core dump file to store the memory status when a process terminates abnormally because of the default action of certain signals.

The signals whose Action is Core in the following figure can cause the Linux operating system to generate core dump files.

![](../images/p107903.png)

## Use the core dump feature in ECI

Before Elastic Container Instance \(ECI\) allows you to specify the naming rule and storage location of core dump files, you cannot customize such information because virtual machines \(VMs\) used by ECI are managed by Alibaba Cloud.

The core dump files are generated based on the default naming rule and stored in the default location.

```
/ # cd /pod/
/pod # ls
data
/pod # ping www.aliyun.com
PING www.aliyun.com (106.11.253.83): 56 data bytes
64 bytes from 106.11.253.83: seq=0 ttl=47 time=25.496 ms
64 bytes from 106.11.253.83: seq=1 ttl=47 time=25.510 ms
64 bytes from 106.11.253.83: seq=2 ttl=47 time=25.640 ms
64 bytes from 106.11.253.83: seq=3 ttl=47 time=25.507 ms
^\Quit (core dumped)
/pod # ls
core.45  data
/pod # 
```

As shown in the preceding command output, the core dump file is named in the core.Process ID format and stored in the working directory of the pod. When an ECI is released, its core dump files are deleted. In this case, you cannot obtain and analyze the generated the core dump files.

ECI allows you to specify the naming rule and storage location of core dump files by setting the CorePattern parameter in an API request. Example: CorePattern = "/xx/xx/core".

```
CorePattern = "/xx/xx/core"
```

Alternatively, you can use an annotation to specify such information in a YAML configuration file of Kubernetes:

```
annotations:
    k8s.aliyun.com/eci-core-pattern: "/xx/xx/core"
```

## Example

If you need to use core dump files of an ECI for offline analysis, you must store the core dump files to an external storage instead of a local directory of an ECI. ECI allows you to store core dump files in a disk or an nfs volume. Object Storage Service \(OSS\) will be supported soon. In this example, core dump files are stored to an nfs volume.

```
'Volume.1.Name': 'default-volume1',
'Volume.1.Type': 'NFSVolume',
'Volume.1.NFSVolume.Path': '/dump/',
'Volume.1.NFSVolume.Server': '143b24822a-gfn3.cn-beijing.nas.aliyuncs.com',

'Container.1.VolumeMount.1.Name': 'default-volume1',
'Container.1.VolumeMount.1.MountPath': '/pod/data/dump/',

'CorePattern':'/pod/data/dump/core-eci',
```

Based on the preceding configuration, the nfs volume is mounted to the /pod/data/dump/ directory of the ECI and the directory for storing core dump files is set to /pod/data/dump/core. In this way, all core dump files generated by the ECI are stored in the /pod/data/dump/ directory and synchronized to the nfs volume. The core dump files are not deleted when the ECI is released. You can still obtain and analyze these core dump files after the ECI is released.

1. Trigger a core dump in a directory of the ECI:

```
/ # ls
bin   dev   etc   home  pod   proc  root  sys   tmp   usr   var
/ # ping www.aliyun.com
PING www.aliyun.com (106.11.253.86): 56 data bytes
64 bytes from 106.11.253.86: seq=0 ttl=47 time=26.351 ms
64 bytes from 106.11.253.86: seq=1 ttl=47 time=26.395 ms
64 bytes from 106.11.253.86: seq=2 ttl=47 time=26.366 ms
64 bytes from 106.11.253.86: seq=3 ttl=47 time=26.378 ms
64 bytes from 106.11.253.86: seq=4 ttl=47 time=26.357 ms
^\Quit (core dumped)
/ # ls
bin   dev   etc   home  pod   proc  root  sys   tmp   usr   var
/ # cd /pod/data/dump/
/pod/data/dump # ls
core-eci.12
```

Based on the preceding command output, the naming rule and directory of core dump files take effect as expected.

2. Release the ECI, log on to another ECI to which the same nfs volume is mounted, and view the core dump file.

```
/ # cd /pod/data/dump/
/pod/data/dump # ls
core-eci.12
```

The core dump file is not deleted with the released ECI. In this case, you can use the core dump file for offline analysis.

For more information about the core dump feature, see [http://man7.org/linux/man-pages/man5/core.5.html](http://man7.org/linux/man-pages/man5/core.5.html).
