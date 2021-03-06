# ifiss code

```matlab
cd('/Users/mxchip/Dev/matlab-dev/pifiss1.0')
```
```matlab
   !/bin/cp ./diffusion/test_problems/unit_rhs.m ./diffusion/specific_rhs.m
   !/bin/cp ./diffusion/test_problems/zero_bc.m ./diffusion/specific_bc.m
   !/bin/cp ./diffusion/test_problems/ref_adiff_edit5.m ./diffusion/specific_adiff.m
```
specific_rhs: 	unit RHS forcing function 应该是右边那个函数 这里我们就沿用 还是ones（nel，1）

specific_bc:	边界条件 zeros(size(xbd))

specific_adiff：	 reference variable diffusion operator 前面的系数的一个函数
```matlab
function [diffx,diffy] = specific_adiff(x,y,nel)
      %ref_adiff   reference variable diffusion operator 
      %   [diffx,diffy] = specific_adiff(x,y,nel);
      %   input
      %          x          x coordinate vector
      %          y          y coordinate vector 
      %          nel        number of elements
      %   PSFEM function: DJS; 23 June 2006. 
      %Copyright (c) 2006 by C.E. Powell and D.J. Silvester (see readme.m)
      diffx =  1*ones(nel,1);  diffy =  1*ones(nel,1); 
      
      div = 10; h = 1/9*ones(3);
      temperture = rand(div*div,1); 
      map1 = reshape(temperture,[div,div]);
      map2 = imfilter(map1, h);
      map3 = imfilter(map2, h);
      temperture = reshape(map3,[div*div,1]);
      
      x_val = -1:(2/div):1;
      y_val = -1:(2/div):1;
      item = 1;
      for i = 1:div
        for j = 1:div
            k = find(x_val(i) < x & x < x_val(i+1) & y_val(j) < y & y < y_val(j+1)); 
            diffx(k) = temperture(item);
            diffy(k) = temperture(item);
            item = item + 1;
        end
      end
      gohome
      cd datafiles
      save diffx_diffy diffx diffy
      fprintf('size(diffx) is %d ,size(diffy) is %d, nel is %d \n ',size(diffx),size(diffy),nel);
      return
```

```matlab
      diffx =  1*ones(nel,1);  diffy =  1*ones(nel,1); 
      k=find(abs(x)<0.5); diffy(k)=0.01; 
      k=find(abs(y)<0.5); diffx(k)=0.01;  
      return
```



## 参数$a$的构建

***
这里我需要做的是随机化的创建数据，这里我的a是是一个模拟数据，使用的是随机化数据，10*10的大小和范围。由于随机化的参数可能导致相邻区域内的值不够有光滑性，所以我就使用滤波器平滑了两次。
***
```matlab
div = 10; h = 1/9*ones(3);
temperture = rand(div*div,1); 
map1 = reshape(temperture,[div,div]);
map2 = imfilter(map1, h);
map3 = imfilter(map2, h);
figure;mesh(map1);
figure;mesh(map2);
figure;mesh(map3)
```
- map1
<img src="pic/filter1.png" style="zoom: 50%;" />
- map2
<img src="pic/filter2.png" style="zoom: 50%;" />
- map3
<img src="pic/filter3.png" style="zoom: 50%;" />

## pifiss 计算初步结果
- map1_U
<img src="pic/map1_U.png" style="zoom: 100%;" />
- map2_U
<img src="pic/map2_U.png" style="zoom: 100%;" />
- map3_U
<img src="pic/map3_U.png" style="zoom: 70%;" />



## 参数a指数增强

- 我觉得这里我可以把我的数据存在文件里，这样可以之后一直调用，有对比性和参考性，要不然每次都随机不太行。
- 上面生成的数据我觉得比较没有起伏感，做出来的U比较像正态分布，所以我希望让它指数增强一下。

```matlab
div = 10;%h = 1/9*ones(3);
%h = [0.1 0.1 0.1;0.1 0.3 0.0;0.1 0.1 0.1];
h = [0.1 0.1 0.1;0.1 0.2 0.1;0.1 0.1 0.1];
temperture = rand(div*div,1); 
map1       = reshape(temperture,[div,div]);
map2       = imfilter(map1, h);
map3       = imfilter(map2, h);
temperture = reshape(map3,[div*div,1]);

map_10     = 10.^map3-1;
map_10     = map_10/max(max(map_10));
map_100    = 100.^map3-1;
map_100    = map_100/max(max(map_100));
map_1000   = 1000.^map3-1;
map_1000   = map_1000/max(max(map_1000));
map_10000  = 10000.^map3-1;
map_10000  = map_10000/max(max(map_10000));

cd generated_data
save maps map1 map2 map3 map_10 map_100 map_1000 map_10000
```

生成了map_10,map_100,map_1000,map_1000。

这些数据更加极端，我希望能看到结果能更加多元，更加偏离正态。

- map_10
<img src="pic/map_10.png" style="zoom: 50%;" />
- map_100
<img src="pic/map_100.png" style="zoom: 50%;" />
- map_1000
<img src="pic/map_1000.png" style="zoom: 50%;" />
- map_10000
<img src="pic/map_10000.png" style="zoom: 50%;" />

## pifiss 计算初步结果

- map_10U
<img src="pic/map_10U.png" style="zoom: 50%;" />
- map_100U
<img src="pic/map_100U.png" style="zoom: 50%;" />
- map_1000U
<img src="pic/map_1000U.png" style="zoom: 50%;" />
- map_10000U
<img src="pic/map_10000U.png" style="zoom: 50%;" />



## 参数选择:      $a$~map_10

最终从图的连续性和鼓包的形状的趋势

map3和map_10都可以作为数据来训练，相差不大，为了让初始数据更加有相异行，那么我们就选择map_10.

```matlab
div = 10;h = 1/9*ones(3);
%h = [0.1 0.1 0.1;0.1 0.2 0.1;0.1 0.1 0.1];
temperture = rand(div*div,1); 
map1       = reshape(temperture,[div,div]);
map2       = imfilter(map1, h);
map3       = imfilter(map2, h);
map_10     = 10.^map3-1;
map_10     = map_10/max(max(map_10));
temperture = reshape(map_10,[div*div,1]);
```

### 数据范例

INPUT：

- $a1$:
<img src="pic/heatmap1.png" style="zoom: 50%;" />
<img src="pic/surfmap1.png" style="zoom: 50%;" />

- $a2$:
<img src="pic/heatmap2.png" style="zoom: 50%;" />
<img src="pic/surfmap2.png" style="zoom: 50%;" />

OUTPUT:

- $U1$:
  <img src="pic/output1.png" style="zoom: 60%;" />

- $U2$:
  <img src="pic/output2.png" style="zoom: 60%;" />

## square_domain.m

grid defining data is saved to the file: square_grid.mat
- compute (x,y) coordinates of vertices
- compute biquadratic element coordinates
- compute boundary vertices and edges

save square_grid.mat mv xy bound mbound grid_type outbc x y

- mv是干啥的？

## p1grid.m

linear element grid generator

## femp1_adiff 

vectorized linear anisotropic diffusion matrix

## 如何制造成对的训练数据呢？

我觉得可以生成很多的数据$a$然后让matlab一行一行读，去运行，输出U。
- xtrain.txt
	shape(1000,100)
	1000行数据 每行数据是10*10的方格数据，
```matlab
data = zeros(1000,100);
for i = 1:1000
    div = 10;h = 1/9*ones(3);
    %h = [0.1 0.1 0.1;0.1 0.2 0.1;0.1 0.1 0.1];
    temperture = rand(div*div,1); 
    map1       = reshape(temperture,[div,div]);
    map2       = imfilter(map1, h);
    map3       = imfilter(map2, h);
    map_10     = 10.^map3-1;
    map_10     = map_10/max(max(map_10));
    temperture = reshape(map_10,[div*div,1]);
    surf(map_10)
    data(i,:)    = roundn(temperture,-2);
end
writematrix(data,'xtrain.txt');
```

- ytrain.txt

  自己写计算的函数，在函数内跑我们的数据 一个循环

  

  现在你需要知道函数的结构 然后自己写一个

  square_adiff是主函数，
  femp1_adiff：
  - 是其中的一个函数，所以我们应该怎么做呢
	- 里面调用的是啥？是specific_adiff.m
	- 
  

```matlab
%square_adiff   solve anisotropic problem in unit square domain 
%    PSFEM scriptfile: DJS; 5 February 2007. 
% Copyright (c) 2007 C.E. Powell, D.J. Silvester
gohome, cd datafiles
save currentsn sn
clear variables
load currentsn
%% define geometry
pde=1; domain=11;
square_domain(2,1)
load square_grid
%
%% set up matrices
qmethod=0;
[evt,eboundt] = p1grid(xy,mv,bound,mbound); 
[A,M,f] = femp1_adiff(xy,evt); 
%这里使用我们的数据a
%% boundary conditions
   [Agal,fgal] = nonzerobc(A,f,xy,bound);
%
%% save resulting system
fprintf('system saved in femlab_adiff.mat ...\n')
gohome
cd datafiles
save femlab_adiff qmethod Agal M  fgal  xy x y 
%
%% compute solution
tic
fprintf('solving linear system ...  ')
x_gal=Agal\fgal;
fprintf('done\n')
etoc=toc; fprintf('Galerkin system solved in  %8.3e seconds\n\n',etoc) 
save femlab_adiff x_gal  -append 
%% plot solution
%if qmethod==2,
solplot(x_gal,xy,x,y,12);
%title(['Q',int2str(qmethod),' solution'])
%drawnow
%end

%% compute a posteriori error estimate
%   [jmp,els] = p1fluxjmps_adiff(x_gal,xy,evt,eboundt);
%   [rhsq,hlsq] = p1res_diff(xy,evt);
%   [err_res,elerr_res] = tdiffpost_res(jmp,els,rhsq,hlsq);
%   save femlab_adiff elerr_res evt  -append 
%   errplot_t(x_gal,elerr_res,evt,xy,x,y,11)
```

  






















