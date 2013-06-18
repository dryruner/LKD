• VFS enables programs to use standard unix system calls to read and write to different filesystems, even on different media. Together, the VFS and the block I/O layer provide the abstractions, interfaces, and glue that allow user-space programs to issue generic system calls to access files via a uniform naming policy on any filesystem, which itself exists on any storage medium.

1. VFS objects and their data structures:
The 4 primary object types of the VFS are:
i)   The superblock object, which represents a specific mounted filesystem.
ii)  The inode object, which represents a specific file.
iii) The dentry object, which represents a directory entry, which is a single component of a path.
iv)  The file object, which represents an open file as associated with a process.

An operations object is contained within each of these primary objects. These objects describe the methods that the kernel invokes against the primary object:
i)   The super_operations object;
ii)  The inode_operations object;
iii) The dentry_operations object;
iv)  The file_operations object.

Each registered filesystem is represented by a file_system_type structure. Each mount point is represented by the vfsmount structure.
Two per-process structures describe the filesystem and files associated with a process. They're the fs_struct structure and file structure.

2. The superblock object:
sb->s_op->write_super(sb); // sb is a pointer to the filesystem's superblock.
struct super_operations is defined in <linux/fs.h>
All the functions in super_operations are invoked by the VFS, in process context. All except dirty_inode() may block if needed.

3. The inode object:
struct inode is defined in <linux/fs.h>. An inode represents each file on a filesystem, but the inode object is constructed in memory only as files are accessed.

4. The dentry object:
A dentry is a specific component in a path. The VFS constructs dentry objects on-the-fly, as needed, when performing directory operations.
Dentry objects are represented by struct dentry and defined in <linux/dcache.h>. Unlike the previous two objects, the dentry object doesn't correspond to any sort of on-disk data structure. The VFS creates it on-the-fly from a string representation of a path name.

• Dentry state:
A valid dentry object can be one of the three states: used, unused, or negative.
i)   A used dentry corresponds to a valid inode (d_inode points to an associated inode) and indicates that there are one or more users of the object (d_count > 0).
ii)  An unused dentry corresponds to a valid inode, but the VFS is not currently using the dentry object (d_count == 0). Because the dentry still points to a valid object, the dentry is kept around - cached - in case it is needed again. If it is necessary to reclaim memory, however, the dentry can be discarded because it is not in active use.
iii) A negative dentry is not associated with a valid inode (d_inode is NULL) because either the inode was deleted or the path name was never correct or begin with. The dentry is kept around, however, so that future lookups are resolved quickly. (Because the failed lookup is expensive, caching the negative results are worthwhile.) Although a negative dentry is useful, it can be destroyed if memory is at a premium because nothing is actually using it.
A dentry object can also be freed, sitting in the slab object cache. In that case, there's no valid reference to the dentry object in any VFS or any filesystem code.

• The dentry cache:
The kernel caches dentry objects in the dentry cache, or simply, the dcache. The dentry cache consists of 3 parts:
i)   Lists of "used" dentries linked off their associated inode via the i_dentry field of the inode object. Because a given inode can have multiple links, there might be multiple dentry objects.
ii)  A doubly linked "least recently used" list of unused and negative dentry objects. The list is inserted at the head, such that entries toward the head of the list are newer.
iii) A hash table and hashing function used to quickly resolve a given path into the associated dentry object.
The dcache also provides the front end to an inode cache, the icache. Inode objects that are associated with dentry objects are not freed because the dentry maintains a positive usage count over the inode. As long as the dentry is cached, the corresponding inodes are cached, too. Consequently, when a path name lookup succeeds from cache, as in the previous example, the associated inodes are already cached in memory.

5. The file object:
The file object is the in-memory representation of an open file. At the same time, there can be multiple file objects in existence for the same file. The object points back to the dentry (which in turn points back to the inode) that actually represents the open file.
Similar to the dentry object, the file object doesn't actually correspond to any on-disk data. 
• File operations:
int ioctl(struct inode* inode, struct file* file, unsigned int cmd, unsigned long arg); 
It is used when the file is an open device node. This function is called from the ioctl() system call. Callers must hold the BKL.
int unlocked_ioctl(struct file* file, unsigned int cmd, unsigned long arg); 
This implements the same functionality as ioctl(), but without needing to hold the BKL. The VFS calls unlocked_ioctl() if it exists in lieu of ioctl() when user-space invokes the ioctl() syscall. Thus filesystems need implement only one, preferably unlocked_ioctl().
int compat_ioctl(struct file* file, unsigned int cmd, unsigned long arg); 
This implements a portal variant of ioctl() for use on 64-bit systems by 32-bit applications. Like unlocked_ioctl(), compat_ioctl() doesn't hold the BKL.

6. Data structures associated with filesystems:
i)  struct file_system_type, defined in <linux/fs.h>, describes the capabilities and behavior of each filesystem. There's only 1 file_system_type structure per file system, regardless of how many instances of the filesystem are mounted on the system, or whether the filesystem is even mounted at all.
ii) struct vfsmount, defined in <linux/mount.h>, represents a specific mount point. When a filesystem is actually mounted, a vfsmount structure is created.

7. Data structures associated with a process:
i)   struct files_struct, defined in <linux/fdtable.h>. This table's address is pointed to by the files entry in the process descriptor. All per-process information about open files and file descriptors is contained therein.
ii)  struct fs_struct, defined in <linux/fs_struct.h>. It contains filesystem information related to a process(like cwd, the root dir of a process) and is pointed at by the fs field in the process descriptor.
iii) struct mnt_namespace, defined in <linux/mnt_namespace.h>. It is pointed to by the mnt_namespace field in the process descriptor. They enable each process to have a unique view of the mounted filesystems on the system - not just a unique root directory.

• For most processes, the process descriptor points to unique files_struct and fs_struct. For processes created with the clone flag CLONE_FILES or CLONE_FS, however, these structures are shared. (Threads usually specify CLONE_FILES and CLONE_FS, and thus, share a single files_struct and fs_struct among themselves. Normal processes, on the other hand, don't specify these flags and consequently have their own filesystems information and open file tables.)
The namespace works the other way. By default, all processes share the same namespace. Only when CLONE_NEWNS flag is specified during clone() is the process given a unique copy of the namespace structure. But most processes do not provide this flag, they inherit their parents' namespaces.
