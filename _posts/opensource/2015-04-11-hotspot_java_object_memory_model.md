---
layout: post
title: Hotspot之Java对象内存布局（三）
category:  源码阅读
tags: [hotspot, openjdk7, java, opensource]
---

这篇文章主要介绍Java实例数据成员，以及类数据成员的内存布局。

本文章节
    
* 目录
{:toc}

Alan Kay在总结了面向对象的语言的几大特征时说，每个对象都是由其他对象所构成的存储。本节就说明这种存储。虚拟机如何通过一个对象引用访问其自身的数据数据成员或类数据成员？其实，虚拟机访问Java的实例数据成员还是类数据成员时，都是一个固定的位置，再加上一个在解析代表Java类的二进制数据时所计算出的偏移：对于实例数据成员，此固定位置就是指向堆空间中的instanceOop；对于类数据成员，此固定位置就是instanceKlass的一个称为mirror的变量。**而偏移就是本章要讲的重点**。Java的数据成员访问和类数据成员访问涉及到的虚拟机指令有getfield、putfield、getstatic、putstatic四条指令。

### 概述
当代表Java类型的二进制数据被类装载器装入时，虚拟机的运行时系统要分析二进制数据，创建对应的实体，也即上一篇所讲的klassOop和instanceKlass的组合对象，来代表此Java类。在分析二进制数据过程中，虚拟机将程序员在Java类中定义的数据成员的元数据都存放在instanceKlass的类型为typeArrayOop的fields字段中，fields是一维数组。这个元数据类型为fieldDescription（TODO），包括数据成员的访问修饰符（如是public还是protected）、类型（int还是引用类型）、偏移（**通过偏移，虚拟机的运行时系统可以访问实例变量或类变量，此时的偏移是无效的，需要计算**），以及一些属性信息。此时，虚拟机就可以计算**偏移**，为Java建立内存布局图，用于后续的对Java实例数据成员或类数据成员的访问。

### 内存布局的计算步骤

1. 在分析代表Java类的二进制数据时，计算Java数据成员的种类和数目：比如静态的引用类型的数目、double和long类型数目、实例变量中属于引用类型的数目、实例变量中的整形变量的数目等等。
2. 确认分配顺序。对于静态变量的分配顺序是确定的：首先oops（即引用类型），然后double,long，然后int，然后short、char，然后byte、boolean；对于实例变量的分配顺序来说，有三种分配方式，三种方式主要差别在与oops的分配时机上（首先分配oops；最后分配oops；结合其父类的oops的位置，再决定本类的oops是先分配还是后分配）。分配方式由虚拟机参数FieldsAllocationStyle决定，下面是hotspot源码关于FieldsAllocationStyle参数的解释：

     > 0 - type based with oops first
     > 1 - with oops last
     > 2 - oops in super and sub classes are together

3. 当所有的，包括静态和实例变量的偏移都计算完成后，更新变量的元数据，即instanceKlass的fields数据成员的偏移信息。至此，内存布局计算完毕。


#### 静态变量内存布局
Hotspot中的静态变量之前是在instanceKlass中分配，现在放到instanceMirrorKlass （instanceKlass中的_java_mirror字段）中进行分配。第一个静态变量是在_java_mirror中的数据成员后面，即偏移为instanceMirrorKlass::offset_of_static_fields()处开始。静态变量的分配顺序是：引用类型(oops)，然后是double、long类型，然后是int/float类型，然后是short、char类型，然后是byte、boolean类型。如下图所示：
![静态变量内存布局]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_memory_java_class.jpg)

#### 实例变量内存布局
1. 计算第一个非静态变量的偏移。偏移是在对象头和其父类的实例变量的后面，即值为instanceOopDesc::base_offset_in_bytes（对象头）与父类的实例变量的和。
2. 决定分配方式。根据CompactFields（是否需要压缩存储）和FieldsAllocationStyle（分配方式）来对分配变量的次序进行控制。分配方式有三种：分别为0、1、2。默认分配方式为1。分配方式的主要区别在于引用类型的变量是先分配还是最后分配。分配方式0是先分配，即oops/double,long/int,float/short,char/byte,boolean，而分配方式1为后分配，即为double,long/int,float/short,char/byte,boolean/oops。分配方式2是根据父类的oop的分布情况选择子类是先分配oops还是最后分配oops：如果父类的最后变量是oops，那么子类就先分配oops，如果不是，那就后分配。
3. 处理压缩存储。比如在64机器上，使用压缩指针时，头部只要12字节，而头部固定为16字节，于是就有4字节的空间剩余，可用于存储Java类的小于等于4字节长度的数据成员，如int型，2字节长度的char型，1字节长度的byte型变量等等。如下图所示：
![实例变量内存布局]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_memory_java_object.jpg)

### Hotspot相关代码
        
        ============vm/classfile/classFileParser.cpp 3124~3165============
        [ClassFileParser::parseClassFile]
        3124        // Field size and offset computation
        3125    int nonstatic_field_size = super_klass()==NULL?0: super_klass->nonstatic_field_size();
        3126     #ifndef PRODUCT
        3127    int orig_nonstatic_field_size =0;
        3128     #endif
        3129    int static_field_size =0;
        3130    int next_static_oop_offset;
        3131    int next_static_double_offset;
        3132    int next_static_word_offset;
        3133    int next_static_short_offset;
        3134    int next_static_byte_offset;
        3135    int next_static_type_offset;
        3136    int next_nonstatic_oop_offset;
        3137    int next_nonstatic_double_offset;
        3138    int next_nonstatic_word_offset;
        3139    int next_nonstatic_short_offset;
        3140    int next_nonstatic_byte_offset;
        3141    int next_nonstatic_type_offset;
        3142    int first_nonstatic_oop_offset;
        3143    int first_nonstatic_field_offset;
        3144    int next_nonstatic_field_offset;
        3145    
        3146    // Calculate the starting byte offsets
        3147         next_static_oop_offset      = instanceMirrorKlass::offset_of_static_fields();
        3148         next_static_double_offset   = next_static_oop_offset +
        3149    (fac.static_oop_count * heapOopSize);
        3150    if( fac.static_double_count &&
        3151    (Universe::field_type_should_be_aligned(T_DOUBLE)||
        3152               Universe::field_type_should_be_aligned(T_LONG))){
        3153           next_static_double_offset = align_size_up(next_static_double_offset, BytesPerLong);
        3154    }
        3155    
        3156    next_static_word_offset= next_static_double_offset +
        3157    (fac.static_double_count * BytesPerLong);
        3158    next_static_short_offset= next_static_word_offset +
        3159    (fac.static_word_count * BytesPerInt);
        3160    next_static_byte_offset= next_static_short_offset +
        3161    (fac.static_short_count * BytesPerShort);
        3162    next_static_type_offset= align_size_up((next_static_byte_offset +
        3163                                       fac.static_byte_count ), wordSize );
        3164    static_field_size=(next_static_type_offset -
        3165                                       next_static_oop_offset)/ wordSize;

代码3125计算父类的实例变量的大小，所有的属于子类的实例变量是存放在父类实例变量的后面。代码3147计算第一个静态变量的偏移。代码3148~3165 按照oops，doubles、longs，ints、floats，shorts、chars，byte、boolean的顺序，计算各个静态变量类型的偏移。


        ============vm/classfile/classFileParser.cpp 3172~3276============
        [ClassFileParser::parseClassFile]
        
        3172     first_nonstatic_field_offset = instanceOopDesc::base_offset_in_bytes()+
        3173                                    nonstatic_field_size * heapOopSize;
        3174     next_nonstatic_field_offset = first_nonstatic_field_offset;
        ......
        3189    unsignedint nonstatic_double_count = fac.nonstatic_double_count;
        3190    unsignedint nonstatic_word_count   = fac.nonstatic_word_count;
        3191    unsignedint nonstatic_short_count  = fac.nonstatic_short_count;
        3192    unsignedint nonstatic_byte_count   = fac.nonstatic_byte_count;
        3193    unsignedint nonstatic_oop_count    = fac.nonstatic_oop_count;
        ......
        3247    bool compact_fields   = CompactFields;
        3248    int  allocation_style = FieldsAllocationStyle;
        3249    if( allocation_style <0|| allocation_style >2){// Out of range?
        3250       assert(false,"0 <= FieldsAllocationStyle <= 2");
        3251       allocation_style =1;// Optimistic
        3252    }
        3253    
        3254    // The next classes have predefined hard-coded fields offsets
        3255    // (see in JavaClasses::compute_hard_coded_offsets()).
        3256    // Use default fields allocation order for them.
        3257    if((allocation_style !=0|| compact_fields )&& class_loader.is_null()&&
        3258    (class_name == vmSymbols::java_lang_AssertionStatusDirectives()||
        3259          class_name == vmSymbols::java_lang_Class()||
        3260          class_name == vmSymbols::java_lang_ClassLoader()||
        3261          class_name == vmSymbols::java_lang_ref_Reference()||
        3262          class_name == vmSymbols::java_lang_ref_SoftReference()||
        3263          class_name == vmSymbols::java_lang_StackTraceElement()||
        3264          class_name == vmSymbols::java_lang_String()||
        3265          class_name == vmSymbols::java_lang_Throwable()||
        3266          class_name == vmSymbols::java_lang_Boolean()||
        3267          class_name == vmSymbols::java_lang_Character()||
        3268          class_name == vmSymbols::java_lang_Float()||
        3269          class_name == vmSymbols::java_lang_Double()||
        3270          class_name == vmSymbols::java_lang_Byte()||
        3271          class_name == vmSymbols::java_lang_Short()||
        3272          class_name == vmSymbols::java_lang_Integer()||
        3273          class_name == vmSymbols::java_lang_Long())){
        3274       allocation_style =0;// Allocate oops first
        3275       compact_fields   =false;// Don't compact fields
        3276    }

代码3172计算第一个非静态变量的偏移，这个偏移是计算对象头和父类的实例变量的和而得到。代码3247确认是否需要压缩存储。代码3248获取Java对象的实例变量的分配方式，默认为1.代码3257~3276 确认是否是一些虚拟机内部的类，这些类的字段分配顺序是硬编码的。

        ============vm/classfile/classFileParser.cpp 3278~3307============
        [ClassFileParser::parseClassFile]
        
        3278    if( allocation_style ==0){
        3279    // Fields order: oops, longs/doubles, ints, shorts/chars, bytes
        3280           next_nonstatic_oop_offset    = next_nonstatic_field_offset;
        3281           next_nonstatic_double_offset = next_nonstatic_oop_offset +
        3282    (nonstatic_oop_count * heapOopSize);
        3283    }elseif( allocation_style ==1){
        3284    // Fields order: longs/doubles, ints, shorts/chars, bytes, oops
        3285           next_nonstatic_double_offset = next_nonstatic_field_offset;
        3286    }elseif( allocation_style ==2){
        3287    // Fields allocation: oops fields in super and sub classes are together.
        3288    if( nonstatic_field_size >0&& super_klass()!=NULL&&
        3289               super_klass->nonstatic_oop_map_size()>0){
        3290    int map_size = super_klass->nonstatic_oop_map_size();
        3291             OopMapBlock* first_map = super_klass->start_of_nonstatic_oop_maps();
        3292             OopMapBlock* last_map = first_map + map_size -1;
        3293    int next_offset = last_map->offset()+(last_map->count()* heapOopSize);
        3294    if(next_offset == next_nonstatic_field_offset){
        3295               allocation_style =0;// allocate oops first
        3296               next_nonstatic_oop_offset    = next_nonstatic_field_offset;
        3297               next_nonstatic_double_offset = next_nonstatic_oop_offset +
        3298    (nonstatic_oop_count * heapOopSize);
        3299    }
        3300    }
        3301    if( allocation_style ==2){
        3302             allocation_style =1;// allocate oops last
        3303             next_nonstatic_double_offset = next_nonstatic_field_offset;
        3304    }
        3305    }else{
        3306           ShouldNotReachHere();
        3307    }

代码3278~3307 根据不同的分配方式，计算引用类型的偏移。除了引用类型外，其它类型的分配顺序是固定的。代码3280~3281计算分配方式1的oops和double、long型变量的偏移。由于默认分配方式是将oops放到最后，所以代码3285只是计算double、long型变量的偏移；在分配方式2中，代码3290~3294首先计算父类对象的尾部是否是oops，如果是，代码3294~3297就在子类中按照分配方式0进行分配，否则代码3302~3303按照分配方式1进行分配。

        ============vm/classfile/classFileParser.cpp 3318~3376============
        [ClassFileParser::parseClassFile]
        3318        if( nonstatic_double_count >0){
        3319    int offset = next_nonstatic_double_offset;
        3320           next_nonstatic_double_offset = align_size_up(offset, BytesPerLong);
        3321    if( compact_fields && offset != next_nonstatic_double_offset ){
        3322    // Allocate available fields into the gap before double field.
        3323    int length = next_nonstatic_double_offset - offset;
        3324             assert(length == BytesPerInt,"");
        3325             nonstatic_word_space_offset = offset;
        3326    if( nonstatic_word_count >0){
        3327               nonstatic_word_count      -=1;
        3328               nonstatic_word_space_count =1;// Only one will fit
        3329               length -= BytesPerInt;
        3330               offset += BytesPerInt;
        3331    }
        3332             nonstatic_short_space_offset = offset;
        3333    while( length >= BytesPerShort && nonstatic_short_count >0){
        3334               nonstatic_short_count       -=1;
        3335               nonstatic_short_space_count +=1;
        3336               length -= BytesPerShort;
        3337               offset += BytesPerShort;
        3338    }
        3339             nonstatic_byte_space_offset = offset;
        3340    while( length >0&& nonstatic_byte_count >0){
        3341               nonstatic_byte_count       -=1;
        3342               nonstatic_byte_space_count +=1;
        3343               length -=1;
        3344    }
        3345    // Allocate oop field in the gap if there are no other fields for that.
        3346             nonstatic_oop_space_offset = offset;
        3347    if( length >= heapOopSize && nonstatic_oop_count >0&&
        3348                 allocation_style !=0){// when oop fields not first
        3349               nonstatic_oop_count      -=1;
        3350               nonstatic_oop_space_count =1;// Only one will fit
        3351               length -= heapOopSize;
        3352               offset += heapOopSize;
        3353    }
        3354    }
        3355    }
        3356    
        3357         next_nonstatic_word_offset  = next_nonstatic_double_offset +
        3358    (nonstatic_double_count * BytesPerLong);
        3359         next_nonstatic_short_offset = next_nonstatic_word_offset +
        3360    (nonstatic_word_count * BytesPerInt);
        3361         next_nonstatic_byte_offset  = next_nonstatic_short_offset +
        3362    (nonstatic_short_count * BytesPerShort);
        3363    
        3364    int notaligned_offset;
        3365    if( allocation_style ==0){
        3366           notaligned_offset = next_nonstatic_byte_offset + nonstatic_byte_count;
        3367    }else{// allocation_style == 1
        3368           next_nonstatic_oop_offset = next_nonstatic_byte_offset + nonstatic_byte_count;
        3369    if( nonstatic_oop_count >0){
        3370             next_nonstatic_oop_offset = align_size_up(next_nonstatic_oop_offset, heapOopSize);
        3371    }
        3372           notaligned_offset = next_nonstatic_oop_offset +(nonstatic_oop_count * heapOopSize);
        3373    }
        3374         next_nonstatic_type_offset = align_size_up(notaligned_offset, heapOopSize );
        3375         nonstatic_field_size = nonstatic_field_size +((next_nonstatic_type_offset
        3376    - first_nonstatic_field_offset)/heapOopSize);

代码3318~3355 处理压缩存储。根据空闲空间的大小，代码3357~3361顺序的处理4字节长度变量、2字节长度变量、1字节长度变量。代码3365~3376 处理引用类型的第一个变量的偏移。

        ============vm/classfile/classFileParser.cpp 3378~3475============
        [ClassFileParser::parseClassFile]
        
        3378    // Iterate over fields again and compute correct offsets.
        3379    // The field allocation type was temporarily stored in the offset slot.
        3380    // oop fields are located before non-oop fields (static and non-static).
        3381    int len = fields->length();
        3382    for(int i =0; i < len; i += instanceKlass::next_offset){
        3383    int real_offset;
        3384           FieldAllocationType atype =(FieldAllocationType) fields->ushort_at(i + instanceKlass::low_offset);
        3385    switch(atype){
        3386    case STATIC_OOP:
        3387               real_offset = next_static_oop_offset;
        3388               next_static_oop_offset += heapOopSize;
        3389    break;
        3390    case STATIC_BYTE:
        3391               real_offset = next_static_byte_offset;
        3392               next_static_byte_offset +=1;
        3393    break;
        3394    case STATIC_SHORT:
        3395               real_offset = next_static_short_offset;
        3396               next_static_short_offset += BytesPerShort;
        3397    break;
        3398    case STATIC_WORD:
        3399               real_offset = next_static_word_offset;
        3400               next_static_word_offset += BytesPerInt;
        3401    break;
        3402    case STATIC_ALIGNED_DOUBLE:
        3403    case STATIC_DOUBLE:
        3404               real_offset = next_static_double_offset;
        3405               next_static_double_offset += BytesPerLong;
        3406    break;
        3407    case NONSTATIC_OOP:
        3408    if( nonstatic_oop_space_count >0){
        3409                 real_offset = nonstatic_oop_space_offset;
        3410                 nonstatic_oop_space_offset += heapOopSize;
        3411                 nonstatic_oop_space_count  -=1;
        3412    }else{
        3413                 real_offset = next_nonstatic_oop_offset;
        3414                 next_nonstatic_oop_offset += heapOopSize;
        3415    }
        3416    // Update oop maps
        ...
        3444    case NONSTATIC_SHORT:
        3445    if( nonstatic_short_space_count >0){
        3446                 real_offset = nonstatic_short_space_offset;
        3447                 nonstatic_short_space_offset += BytesPerShort;
        3448                 nonstatic_short_space_count  -=1;
        3449    }else{
        3450                 real_offset = next_nonstatic_short_offset;
        3451                 next_nonstatic_short_offset += BytesPerShort;
        3452    }
        3453    break;
        3454    case NONSTATIC_WORD:
        3455    if( nonstatic_word_space_count >0){
        3456                 real_offset = nonstatic_word_space_offset;
        3457                 nonstatic_word_space_offset += BytesPerInt;
        3458                 nonstatic_word_space_count  -=1;
        3459    }else{
        3460                 real_offset = next_nonstatic_word_offset;
        3461                 next_nonstatic_word_offset += BytesPerInt;
        3462    }
        3463    break;
        3464    case NONSTATIC_ALIGNED_DOUBLE:
        3465    case NONSTATIC_DOUBLE:
        3466               real_offset = next_nonstatic_double_offset;
        3467               next_nonstatic_double_offset += BytesPerLong;
        3468    break;
        3469    default:
        3470               ShouldNotReachHere();
        3471    }
        3472           fields->short_at_put(i + instanceKlass::low_offset,  extract_low_short_from_int(real_offset));
        3473           fields->short_at_put(i + instanceKlass::high_offset, extract_high_short_from_int(real_offset));
        3474    }
        3475    

代码3378~3475 依据各种类型字段的第一个变量的偏移，顺次计算此种类的其他变量的偏移，并更新变量的元数据信息。至此，Java内存布局计算完毕。

### 实践
Java代码：

        class MemoryLayoutDefault {
        byte a =(byte)0xab;
        int c =0x2222;
        boolean d =true;
        long e =0xbadbeef;
            String _string ="start";
            Integer _int =0x1111;
            Long _long =0x12345678l;
            String _string2 ="end";
        }
        class SubMemoryLayout extends MemoryLayoutDefault {
        int sub_c =0x3333;
        long sub_e =0xbeebee;
            Integer _sub_int =0x4444;
            Long _sub_long =0x12345678l;
        }

使用hsdb进行验证：

>**hsdb>**`universe`
Heap Parameters:
Gen 0:   eden [0x00000000bee00000,0x00000000beea4828,0x00000000bfe10000) space capacity = 16842752, 4.000723872203308 used
  from [0x00000000bfe10000,0x00000000bfe10000,0x00000000c0000000) space capacity = 2031616, 0.0 used
  to   [0x00000000c0000000,0x00000000c0000000,0x00000000c01f0000) space capacity = 2031616, 0.0 usedInvocations: 0
Gen 1:   old  [0x00000000d2e00000,0x00000000d2e00000,0x00000000d5600000) space capacity = 41943040, 0.0 usedInvocations: 0
  perm [0x00000000fae00000,0x00000000fb070568,0x00000000fc2c0000) space capacity = 21757952, 11.753348844597138 usedInvocations: 0
**hsdb**>`scanoops 0x00000000bee00000 0x00000000bfe10000 MemoryLayoutDefault`
0x00000000bee4f580 MemoryLayoutDefault
0x00000000bee52200 SubMemoryLayout
**hsdb>**`inspect 0x00000000bee4f580`
instance of Oop for MemoryLayoutDefault @ 0x00000000bee4f580 @ 0x00000000bee4f580 (size = 48)
_mark: 5
a: 171
c: 8738
d: true
e: 195935983
_string: "start" @ 0x00000000bee4f5b0 Oop for java/lang/String @ 0x00000000bee4f5b0
_int: Oop for java/lang/Integer @ 0x00000000bee50d38 Oop for java/lang/Integer @ 0x00000000bee50d38
_long: Oop for java/lang/Long @ 0x00000000bee50f08 Oop for java/lang/Long @ 0x00000000bee50f08
_string2: "end" @ 0x00000000bee50f20 Oop for java/lang/String @ 0x00000000bee50f20
**hsdb>**`mem 0x00000000bee4f580 6`
0x00000000bee4f580: 0x0000000000000005 
0x00000000bee4f588: **0x00002222**fb06ed80 
0x00000000bee4f590:**0x000000000badbeef** 
0x00000000bee4f598: **0xbee4f5b0000001ab**
0x00000000bee4f5a0: **0xbee50f08bee50d38** 
>0x00000000bee4f5a8: 0x00000000**bee50f20**

默认的情况下，指针在64位上是压缩存储，即占4字节。同时，组成Java对象的数据成员是压缩存储。默认的Java对象中变量分配方式为：long、double，int、float，short、char，byte、boolean，oops的方式存储。
首先是对象头占12字节（因为CompressedOops默认为true），由于是压缩存储，且存在8字节长度的变量（即存在long或者double型的变量），所以需要对数据成员进行压缩处理。压缩处理首先处理4字节长度的变量，这里就是int型变量0x2222；接着分配的是8字节长度的变量――long型的0xbadbeef。接着处理4字节长度和2字节长度变量，这里没有，所以接着处理1字节长度变量――byte型0xab和boolean型变量true。在处理完成后，接着分配oops，oop是4字节对齐，所以oop从4字节的整数倍开始存储，分别为0xbee4f5b（String变量_string）、0x bee50d38(Integer型变量_int)、0xbee50f08（Long型变量_long）、0xbee50f20（String型变量_string2）。

>**hsdb>** `inspect 0x00000000bee52200`(此处看一下继承情况下的Java对象的内存布局，此对象为SubMemoryLayout，继承至MemoryLayoutDefault)
instance of Oop for SubMemoryLayout @ 0x00000000bee52200 @ 0x00000000bee52200 (size = 64)
_mark: 5
a: 171
c: 8738
d: true
e: 195935983
_string: "start" @ 0x00000000bee4f5b0 Oop for java/lang/String @ 0x00000000bee4f5b0
_int: Oop for java/lang/Integer @ 0x00000000bee52240 Oop for java/lang/Integer @ 0x00000000bee52240
_long: Oop for java/lang/Long @ 0x00000000bee52250 Oop for java/lang/Long @ 0x00000000bee52250
_string2: "end" @ 0x00000000bee50f20 Oop for java/lang/String @ 0x00000000bee50f20
sub_c: 13107
sub_e: 12512238
_sub_int: Oop for java/lang/Integer @ 0x00000000bee52268 Oop for java/lang/Integer @ 0x00000000bee52268
_sub_long: Oop for java/lang/Long @ 0x00000000bee52278 Oop for java/lang/Long @ 0x00000000bee52278
**hsdb>** `mem 0x00000000bee52200 8`
0x00000000bee52200: **0x00000000**00000005 
0x00000000bee52208: **0x00002222fb070208** 
0x00000000bee52210: **0x000000000badbeef**
0x00000000bee52218: **0xbee4f5b0000001ab** 
0x00000000bee52220: 0xbee52250**bee52240** 
0x00000000bee52228: 0x00003333bee50f20
0x00000000bee52230: 0x0000000000beebee 
>0x00000000bee52238: 0xbee52278bee52268

粗体的是SubMemoryLayout的父类的实例变量，接着才是SubMemoryLayout的实例变量。由于压缩存储且存在8字节变量，故而存在由于对齐而空出的空间，所以可在空出的空间中存放4字节长度的变量（或长度小于4字节长度的变量），这里是int型变量0x3333。处理完压缩空间后，接着存放一个long型变量，然后是两个oop变量。

>**hsdb>**`universe`
Heap Parameters:
Gen 0:   eden [0x00000000bee00000,0x00000000beea4828,0x00000000bfe10000) space capacity = 16842752, 4.000723872203308 used
  from [0x00000000bfe10000,0x00000000bfe10000,0x00000000c0000000) space capacity = 2031616, 0.0 used
  to   [0x00000000c0000000,0x00000000c0000000,0x00000000c01f0000) space capacity = 2031616, 0.0 usedInvocations: 0
Gen 1:   old  [0x00000000d2e00000,0x00000000d2e00000,0x00000000d5600000) space capacity = 41943040, 0.0 usedInvocations: 0
  perm [0x00000000fae00000,0x00000000fb070458,0x00000000fc2c0000) space capacity = 21757952, 11.752098726939005 usedInvocations: 0
**hsdb>**`scanoops 0x00000000bee00000 0x00000000bfe10000 SubMemoryLayout`
0x00000000bee52250 **SubMemoryLayout**
hsdb>inspect 0x00000000bee52250
instance of Oop for SubMemoryLayout @ 0x00000000bee52250 @ 0x00000000bee52250 (size = 64)
_mark: 5
a: 171
c: 8738
d: true
e: 195935983
_string: "start" @ 0x00000000bee4f600 Oop for java/lang/String @ 0x00000000bee4f600
_int: Oop for java/lang/Integer @ 0x00000000bee52290 Oop for java/lang/Integer @ 0x00000000bee52290
_long: Oop for java/lang/Long @ 0x00000000bee522a0 Oop for java/lang/Long @ 0x00000000bee522a0
_string2: "end" @ 0x00000000bee50f70 Oop for java/lang/String @ 0x00000000bee50f70
sub_c: 13107
sub_e: 12512238
_sub_int: Oop for java/lang/Integer @ 0x00000000bee522b8 Oop for java/lang/Integer @ 0x00000000bee522b8
_sub_long: Oop for java/lang/Long @ 0x00000000bee522c8 Oop for java/lang/Long @ 0x00000000bee522c8
**hsdb>**`mem 0x00000000bee52250 8`
0x00000000bee52250: 0x0000000000000005 
0x00000000bee52258: 0x00002222fb070100 
0x00000000bee52260: 0x000000000badbeef 
0x00000000bee52268: 0xbee4f600000001ab 
0x00000000bee52270: 0xbee522a0bee52290 
0x00000000bee52278: **0xbee522b8**bee50f70 
0x00000000bee52280: 0x00003333**bee522c8**
>0x00000000bee52288: 0x0000000000beebee

启动jdb时添加-XX:FieldsAllocationStyle=2，使用分配方式2对Java对象的分配方式进行分配。注意在0x00000000bee52278+0x8处的值为0xbee522b8为SubMemoryLayout中的实例变量_sub_int，即将父类的oops和子类的oops放在一起。

  