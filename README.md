　　部门的产品使用自己公司研发的系统，嵌入式web服务器移植的是goahead2.5的，服务器和前端使用JSON交互，移植的cJSON库，所以这段时间对JSON的使用做个简单的笔记，cJSON.h提供出来的接口很多，但是感觉平时使用的也就那么几个。

　　在做测试的时候，通过创建json对象，添加节点，然后保存，读取，输出这样的一个流程，发现当添加节点数多的时候，会会出现长时间的等待，当时好像是一万行的数据量，整个创建过程花费了2，3秒钟，所以当更多数据量的时候，花费的时间可能更长。最后发现是这个函数导致的结果，源码如下，可以看到它每添加一个item，都是从头往后找，等找到最后一个节点的时候，然后把item赋值给最后一个节点的next，所以节点越多，时间也就更长了。

void   cJSON_AddItemToArray(cJSON *array, cJSON *item)

void   cJSON_AddItemToArray(cJSON *array, cJSON *item)
{
    cJSON *c=array->child;
    if (!item) return;
     if (!c) 
    {
        array->child=item;
    } 
    else
     {
        while (c && c->next) 
        c=c->next; 
        suffix_object(c,item);
    }
}                
　　查看cJSON的结构体，会发现，json结构有next和pre两个指针，也就是它的链表是个双向链表，但是就奇怪为何找节点却不用这个优点，非得单向去找。

typedef struct cJSON {
    struct cJSON *next,*prev;    /* next/prev allow you to walk array/object chains. Alternatively, use GetArraySize/GetArrayItem/GetObjectItem */
    struct cJSON *child;        /* An array or object item will have a child pointer pointing to a chain of the items in the array/object. */

    int type;                    /* The type of the item, as above. */

    char *valuestring;            /* The item's string, if type==cJSON_String */
    int valueint;                /* The item's number, if type==cJSON_Number */
    double valuedouble;            /* The item's number, if type==cJSON_Number */

    char *string;                /* The item's name string, if this item is the child of, or is in the list of subitems of an object. */
} cJSON;
　　所以解决的思路就在这了，有两种方式解决：

　　1，利用array的pre指针，每次插入item后，同时将其指针保存在array->child->pre中，这样我每次插入节点，都只需要找到第一个节点的pre指针，然后将item插到该地址之后，即可。

　　cJSON * c = array->child;
    if(!item)
    {
        return ;
    }
    if(!c)
    {
        array->child = item;
        array->child->prev = item;
    }
    else
    {
        array->child->prev->next = item;
        array->child->prev = item;
        item->next = NULL;
　　}
　　2，第二种方式就很简单，通过修改json结构体实现目的，在结构体中添加一个成员 struct cJSON * last;每次添加item的时候，同时将它的指针赋值给array->child->last;

这样每次添加的时候，只需要查找last指针就可以找到最后一个节点。

　　cJSON * c = array->child;
    if(!item)
    {
        return ;
    }
    if(!c)
    {
        array->child = item;
        array->child->last = item;
    }
    else
    {
        array->child->last->next = item;
        array->child->last = item;
        item = NULL;
    }
　　用的最多的object对象就是这些了。

#define cJSON_AddNullToObject(object,name) 
#define cJSON_AddTrueToObject(object,name) 
#define cJSON_AddFalseToObject(object,name) 
#define cJSON_AddBoolToObject(object,name,b)
#define cJSON_AddNumberToObject(object,name,n) 
#define cJSON_AddStringToObject(object,name,s) 

还有数组对象

void cJSON_AddItemToArray(cJSON *array, cJSON *item);
void	cJSON_AddItemToObject(cJSON *object,const char *string,cJSON *item);

如果是将一个数组添加进对象就可以用

void cJSON_AddItemReferenceToArray(cJSON *array, cJSON *item);

void cJSON_AddItemReferenceToObject(cJSON *object,const char *string,cJSON *item);

当用完json对象时候，就必须记者删除 

cJSON_Delete(cJSON*);

将一个字符串解析成json对象

extern cJSON *cJSON_Parse(const char *value);

将一个json对象转换成char *，但是这个字符串必须是手动删除

extern char  *cJSON_Print(cJSON *item);

 