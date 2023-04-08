# 前言1

众所周知，angr有非常多的使用限制，一般的程序很难直接使用angr来进行解题。但是，虽然不能直接使用angr来进行解题，但是我们完全可以间接使用angr。一般的程序，只要知道它的加密逻辑后，我们就可以用C语言复现这个程序，保证同样的输入下，重写的程序和原程序其加密之后的结果是一样的，然后再用angr对重写的程序进行解题。

# 这种解题方法的优势与劣势

## 优势：

我们可以不用再完全分析出程序的加密逻辑，也不用知道加密算法的逆算法是怎么样的，我们只需要无脑重写程序的逻辑，无脑使用angr即可。

## 劣势：

这种解题方法很显然存在缺陷，很容易看到，在重写程序这一步骤，我们需要花费大量时间来保证加密逻辑与原程序是完全一致的，往往在短时间内很难一次就复现成功，这对我们的解题信心会产生很大的冲击。

# 前言2

这种解题方法在之前学angr的时候就已经想过，但是奈何没有合适的题目来进行实践。前几天KM爷给了我们一道raffle，在做这道raffle的时候，想过其他的解法，一种是按照加密函数的顺序一步步逆回去，但是这个方法破产了，因为有的函数无法逆，因为在加密中进行了这样的操作：```main_input[3]*=54```，其中这个输入的数组是以字节为基本元素的，很显然无法逆回去。于是我就想到了angr。angr无疑是非常强大的，但是限制也非常多，很多时候，根本不知道angr会在哪一个库函数进去出不来了，在这道题中在angr中实现hook也非常不容易或者说非常困难。这个程序不能用angr但是我又非常想用angr，怎么办？那干脆我重写一个程序吧。重写一个和原来的程序加密逻辑一样的程序，这样再用angr跑，怎么样？



# 分析这道题目

在分析这道题目的同时，我们会学到一些go 语言逆向的知识。

```c++
if ( &v29 <= *(__readfsqword(0xFFFFFFF8) + 16) )
    runtime_morestack_noctxt(v1);
```

这段代码的意思是如果栈空间不够用了，那么就延长一下栈空间。

```C++
 if ( qword_566628 >= 2 && *(os_Args + 24) == 45LL )
```

这个判断语句的意思是，命令行参数大于等于2，且第二个参数的字符数量等于45

这个代码就告诉了我们，这个程序时从命令行参数读取输入的。

由于笔者没有学过相关知识，无法解释这句代码，这段代码的具体实现功能是通过动态调试推测出来的。

```c++
sync___WaitGroup__Add(v1, v2, v3);
    runtime_newproc(v1, v2, v4, v5, v6, v7, 0, off_4CDC68);
    runtime_newproc(v1, v2, v8, v9, v10, v11, 0, off_4CDBC0);
    runtime_newproc(v1, v2, v12, v13, v14, v15, 0, off_4CDBD0);
    v23 = &main_wg;
    sync___WaitGroup__Wait();
```

这段代码是Go语言的代码，大致的意思是：

首先，调用了一个名为`sync___WaitGroup__Add`的函数，向`WaitGroup`中添加了三个值为`v1`、`v2`、`v3`的任务。

接着，使用`runtime_newproc`函数创建了三个新的进程来执行这三个任务。

最后，使用`sync___WaitGroup__Wait`函数等待这三个任务完成。在等待期间，程序将被阻塞，直到这三个任务全部完成为止。`v23`变量指向了一个名为`main_wg`的`WaitGroup`对象，用于同步这三个任务的完成。



上面的代码就是这个程序的关键加密代码了。

上面三个加密函数中每个函数又分别包括了三个函数，而这三个加密函数中又包括三个加密函数。

总共是39个加密函数。

通过动态调试可以知道这39个加密函数的执行顺序。



#  用C语言重写程序

源代码：

```C
#include<stdio.h>
unsigned char d[]={ //密文
  0x22, 0x3E, 0xBA, 0x05, 0x43, 0xC8, 0x13, 0xE2, 0xDC, 0x5D, 
  0xC6, 0xF2, 0x99, 0x2F, 0xA3, 0xB2, 0x43, 0xFD, 0x4F, 0x60, 
  0x9D, 0x71, 0x9B, 0x0C, 0xDE, 0x98, 0xA2, 0x23, 0xCB, 0x7D, 
  0x80, 0xC1, 0xD3, 0x1E, 0x45, 0x90, 0x5D, 0xA8, 0xE0, 0x72, 
  0xF5, 0x2F, 0x13, 0xA3, 0xDD
};
unsigned char k[46];
void sub_29(){
	k[6]^=k[17];
	k[23]^=k[10];
	k[27]^=k[41];
}
void sub_31(){
	k[26]+=51;
	k[13]^=k[44];
	k[8]-=105;
}
void sub_28(){
	k[10]+=14;
	k[38]*=49;
	k[0x29]^=k[0x2B];
}
void sub_1(){
	k[39]+=26;
	k[14]+=78;
	k[8]^=k[11];

}


void sub_2(){
	k[36]-=66;
	k[35]+=12;	 
	k[16]*=87;
}

void sub_3(){
	k[7]^=k[19];
	k[6]*=27;
	k[13]-=40;
}
void sub_4(){
	k[21]*=9;
	k[25]*=-91;
	k[41]+=45;
}
void sub_5(){
	k[9]^=k[0];
	k[44]*=29;
	k[10]*=-47;
}
void sub_6(){
	k[20]*=-9;
	k[5]-=75;
	k[1]^=k[44];
}




void sub_33(){
	k[9]-=35;
	k[20]+=122;
	k[29]*=123;
}
void sub_34(){
	k[34]^=k[19];
	k[28]*=101;
	k[37]*=61;	
}
void sub_20(){
	k[28]^=k[26];
	k[16]^=k[6];
	k[3]-=29;
}
void sub_7(){
	k[22]*=87;
	k[33]-=57;
	k[16]-=69;

}





void sub_8(){
	k[13]-=51;
	k[33]-=119;
	k[31]^=k[18];
}



void sub_39(){
	k[33]*=33;
	k[21]+=103;
	k[7]*=-77;
}
void sub_15(){
	k[22]^=k[5];
	k[21]^=k[4];
	k[14]*=-47;
}
void sub_14(){
	k[17]*=29;
	k[44]^=k[19];
	k[27]^=k[1];
}
void sub_9(){
	k[40]*=109;
	k[32]-=3;
	k[35]^=k[42];

} 

void sub_10(){
	k[0]+=12;
	k[10]-=118;
	k[19]*=103;
}

void sub_11(){
	k[30]+=83;
	k[11]^=k[41];
	k[38]^=k[22];
}
void sub_12(){
	k[6]-=87;
	k[0]^=k[33];
	k[15]^=k[8];
}




void sub_30(){
	k[0]*=-53;
	k[40]^=k[0];
	k[18]+=79;
}
void sub_24(){
	k[5]-=111;
	k[42]-=65;
	k[1]*=91;
}
void sub_22(){
	k[36]^=k[1];
	k[24]+=36;
	k[2]-=118;
}
void sub_18(){
	k[38]*=35;
	k[8]*=-71;
	k[22]-=48;

}



void sub_37(){
	k[30]^=k[11];
	k[5]*=-29;
	k[19]+=58;
} 
void sub_21(){
	k[3]^=k[0];
	k[14]^=k[29];
	k[7]+=39;
}
void sub_25(){
	k[31]*=-29;
	k[2]^=k[26];
	k[37]^=k[0];

}
void sub_13(){
	k[1]-=108;
	k[42]*=125;
	k[26]*=-15;

}
void sub_16(){
	k[5]+=67;
	k[42]^=k[4];
	k[30]*=85;
}


void sub_26(){
	k[5]-=71;
	k[32]*=-51;
	k[3]*=85;

}

void sub_27(){
	k[28]^=k[6];
	k[31]+=29;
	k[5]^=k[7];
}
void sub_23(){
	k[34]+=78;
	k[20]^=k[8];
	k[38]+=3;

}

void sub_35(){
	k[42]-=88;
	k[31]+=13;
	k[12]*=119;
}
void sub_19(){
	k[36]*=21;
	k[43]^=k[35];
	k[40]+=120;
}
void sub_32(){
	k[12]^=k[35];
	k[34]*=99;
	k[18]^=k[19];

}

void sub_17(){
	k[27]*=11;
	k[24]*=-81;
	k[25]^=k[19];
	
}
void sub_36(){
	k[4]^=k[20];
	k[4]*=-101;
	k[25]-=104;

}
void sub_38(){
	k[39]^=k[36];
	k[9]*=-35;
	k[11]^=k[24];

}

int main(){
	scanf("%s",&k);
	sub_17();
	sub_26();
	sub_16();
	sub_38();
	sub_7();
	sub_33();
	sub_13();
	sub_1();
	sub_28();
	sub_32();
	
	sub_5();
	sub_23();
	sub_27();
	sub_12();
	sub_2();
	sub_36();
	sub_4();
	sub_9();
	sub_39();
	sub_20();
	
	sub_34();
	sub_25();
	sub_37();
	sub_18();
	sub_30();
	sub_29();
	sub_31();
	sub_19();
	sub_35();
	sub_10();
	
	sub_3();
	sub_11();
	sub_8();
	sub_14();
	sub_15();
	sub_6();
	sub_21();
	sub_22();
	sub_24();
	/*
	for(int i=0;i<4;i++){
		for(int j=0;j<10;j++){
			printf("0x%x ",k[i*10+j]);
		}
		printf("\n");
	}
	for(int i=40;i<45;i++){
		printf("0x%x ",k[i]);
	}*/
	if(
	d[0]==k[0]&&d[1]==k[1]&&d[2]==k[2]&&d[3]==k[3]&&d[4]==k[4]&&d[5]==k[5]&&d[6]==k[6]&&d[7]==k[7]&&d[8]==k[8]&&d[9]==k[9]&&
	d[10]==k[10]&&d[11]==k[11]&&d[12]==k[12]&&d[13]==k[13]&&d[14]==k[14]&&d[15]==k[15]&&d[16]==k[16]&&d[17]==k[17]&&d[18]==k[18]&&d[19]==k[19]&&
	d[20]==k[20]&&d[21]==k[21]&&d[22]==k[22]&&d[23]==k[23]&&d[24]==k[24]&&d[25]==k[25]&&d[26]==k[26]&&d[27]==k[27]&&d[28]==k[28]&&d[29]==k[29]&&
	d[30]==k[30]&&d[31]==k[31]&&d[32]==k[32]&&d[33]==k[33]&&d[34]==k[34]&&d[35]==k[35]&&d[36]==k[36]&&d[37]==k[37]&&d[38]==k[38]&&d[39]==k[39]&&
	d[40]==k[40]&&d[41]==k[41]&&d[42]==k[42]&&d[43]==k[43]&&d[44]==k[44]
	){
		printf("you are right!");
		
	}
	else{
		printf("no!");
	}
	
	
}
  /*
  这里是当输入是123456789012345678901234567890123456789012345时加密后的结果
  0x3A, 0x7B, 0x7E, 0x1D, 0x5A, 0x24, 0xC8, 0xB5, 0x47, 0x7C, 
  0xF9, 0x3A, 0x53, 0xDA, 0xE3, 0x06, 0xDE, 0x58, 0x58, 0xA6, 
  0x8D, 0x73, 0xFB, 0x5B, 0x5F, 0xD6, 0xFA, 0x9D, 0x4D, 0x10, 
  0x9F, 0x42, 0xD4, 0xE4, 0xFD, 0xAC, 0x78, 0x92, 0x61, 0x21, 
  0x6B, 0x33, 0x55, 0x98, 0xA7
  */
```

这里为了让angr能够正常执行，我们在最后加密后的数据与密文比较时，不要使用for循环，而是一个一个的比较，然后全部用&&连接。

# angr代码

用angr运行就很简单了

```python
import angr
import claripy
path="C:\\Users\\DELL\\Desktop\\_raffle.exe"
project=angr.Project(path,auto_load_libs=False)
start_addr=0x401F4A
initial_state=project.factory.blank_state(addr=start_addr)

passwordsize=45*8
password=claripy.BVS('password',passwordsize)
password_addr=0x408040
initial_state.memory.store(password_addr,password)

simulation=project.factory.simulation_manager(initial_state)

simulation.explore(find=0x4023CF,avoid=0x4023DD)

if simulation.found:
    for i in simulation.found:
        solution_state=i
        solution = solution_state.solver.eval(password, cast_to=bytes)
        print(solution)
else :
    print("no!")
    # w3_4r3_901ng_70_raffl3_0fF_A_g0...g0-r0u71n3!
```







