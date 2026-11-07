---
layout:     post
title:      Linux 内核之旅（三十二）：内核路径解析 d_path 的实现分析
subtitle:   d_path && bpf_d_path
date:       2026-10-30
author:     pandaychen
catalog:    true
tags:
    - Kernel
    - eBPF
---


##  0x00    前言

`d_path` 是内核提供的基于 dentry 和 vfsmount 获取绝对路径函数

```c
extern char *d_path(const struct path *, char *, int);
```
##  0x01    d_path 的实现分析

####	并发分析

##	0x02	内核中对 d_path 的调用场景

```c
//https://elixir.bootlin.com/linux/v5.4.241/source/fs/seq_file.c#L474
int seq_path(struct seq_file *m, const struct path *path, const char *esc)
{
	char *buf;
	size_t size = seq_get_buf(m, &buf);
	int res = -1;

	if (size) {
		char *p = d_path(path, buf, size);
		if (!IS_ERR(p)) {
			char *end = mangle_path(buf, p, esc);
			if (end)
				res = end - buf;
		}
	}
	seq_commit(m, res);

	return res;
}
EXPORT_SYMBOL(seq_path);
```

##  0x0 参考
-	[Helper function bpf_d_path](https://github.com/isovalent/ebpf-docs/blob/master/docs/linux/helper-function/bpf_d_path.md)