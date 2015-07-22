##Digging the WebCore Mach-O file

使用otool 和 nm 工具可以查看Mach-O 文件的内容。配合不一样的参数，来获取不一样的内容。但是，我们知道在iOS 平台上是用不了otool 和 nm 命令的。如果你想在iOS 平台下分析Mach-O 文件，那么你可以继续往下看。：）

Here we good...

---
####step 1
要获取Mach-O 文件的内容，首先，我们要获取文件的Mach Header

```
- (const struct mach_header_t *)getMachHeaderWithFrameworkName:(NSString *)frameworkName
{
    if ([frameworkName length] == 0)
    {
        return NULL;
    }
    const char* targetName = [frameworkName UTF8String];
    size_t targetNameLength = strlen(targetName);
    uint32_t count = _dyld_image_count();
    const struct mach_header_t *machHeader = NULL;
    
    for(uint32_t dyldIndex = 0; dyldIndex < count; dyldIndex++)
    {
        //Name of image (includes full path)
        const char *dyld = _dyld_get_image_name(dyldIndex);
        printf("%s\n",dyld);
        
        
        //Get name of file
        int length = (int)strlen(dyld);
        int charIndex = length - 1;
        for(; charIndex>= 0; --charIndex)
        {
            if(dyld[charIndex] == '/')
            {
                break;
            }
        }
        charIndex++;
        char *name = strndup(dyld + charIndex, length - charIndex);
        if (NULL != name && strlen(name) == strlen([frameworkName UTF8String]))
        {
            if (strncmp(targetName, name, targetNameLength) == 0)
            {
                machHeader = (const struct mach_header_t *)_dyld_get_image_header(dyldIndex);
                self.slide = _dyld_get_image_vmaddr_slide(dyldIndex);
                free(name);
                break;
            }
            free(name);
        }
    }
    return machHeader;
}

```

Pretty easy ,So go to step 2.

---

####Step 2

我们知道Mach-O 的文件结构如下图：

![](./mach_o_segments.gif)

Header 拿到了，接下来就是要获取Segment 了。

```
struct segment_command_t *find_segment(struct mach_header_t *mh, const char *segname)
{
    struct load_command *lc;
    struct segment_command_t *seg, *foundseg = NULL;
    
    lc = (struct load_command *)((uint64_t)mh + sizeof(struct mach_header_t));
    while ((uint64_t)lc < (uint64_t)mh + (uint64_t)mh->sizeofcmds) {
        if (lc->cmd == LC_SEGMENT_t) {
            seg = (struct segment_command_t *)lc;
            if (strcmp(seg->segname, segname) == 0) {
                foundseg = seg;
                break;
            }
        }
        lc = (struct load_command *)((uint64_t)lc + (uint64_t)lc->cmdsize);
    }
    
    return foundseg;
}

```

Segment 拿到了，还等什么，获取对应的Section 吧，Go to Step 3

---
####Step 3

获取对应Segment 的 Section

```
struct section_t *find_section(struct segment_command_t *seg, const char *name)
{
    struct section_t *sect, *foundsect = NULL;
    u_int i = 0;
    sect = (struct section_t *)((uint64_t)seg + (uint64_t)sizeof(struct segment_command_t));
    for (i = 0;i < seg->nsects;i++)
    {
        if (strcmp(sect->sectname, name) == 0) {
            foundsect = sect;
            break;
        }
        sect = (struct section_t *)((uint64_t)sect + sizeof(struct section_t));
    }
    return foundsect;
}

```

现在Segment 和 Section 都可以拿到了，但是都没有看到任何与符号相关的东西。客官别急，马上来。Go to step 4.

---
####Step 4

要打印符号，首先要找到 LINKEDIT Segment 和 Symtab command.LINKEDIT Segment 可以用 Step 2 的方法获取，Symtab command 可用下面的方法

linkedit = find_segment(machHeader, SEG_LINKEDIT);

symCommand = (struct symtab_command *)find_load_command(machHeader, LC_SYMTAB);

```
struct load_command *find_load_command(struct mach_header_t *mh, uint32_t cmd)
{
    struct load_command *lc, *foundlc;
    lc = (struct load_command *)((uint64_t)mh + sizeof(struct mach_header_t));
    while ((uint64_t)lc < (uint64_t)mh + (uint64_t)mh->sizeofcmds)
    {
        if (lc->cmd == cmd) {
            foundlc = (struct load_command *)lc;
            break;
        }
        lc = (struct load_command *)((uint64_t)lc + (uint64_t)lc->cmdsize);
    }
    
    return foundlc;
}

```

接着，通过公式：

linkEditBase = (uint8_t *)(slide + linkedit->vmaddr - linkedit->fileoff);

strtab = (const char *)(&linkEditBase[symCommand->stroff]);
    
    
symTab = (struct nlist_t *)(&linkEditBase[symCommand->symoff]);


最后就是打印符号了

```
struct nlist_t *nl = NULL;
nl = (struct nlist_t *)symTab;
for (i = 0;i < symCommand->nsyms;i++)
{
    str = (char *)strtab + nl->n_un.n_strx;
    
    if (strcmp(str, name) == 0) {
        addr = (void *)nl->n_value;
        printf("Symbol name :%s\n",name);
        printf("Address :%p\n",addr);
        break;
    }
    nl = (struct nlist_t *)((uint64_t)nl + sizeof(struct nlist_t));
}
```

详情可以查看[Demo](https://github.com/vedon/Mach-O/blob/master/Mach-O%20WebCore/WebCore_Mach-O.zip)