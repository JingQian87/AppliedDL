### NOTE OF PROJECT-ADL

* 没有办法用pickle dump model, 与程序矛盾。查别的办法？

* sample label 1 class 太慢了，可以直接从tumor pixel中sample么

* 决定取level3作为high resolution, level4作为lower resolution. 取level3上的128*128 作为checking tumor region, 这样level4上就是predict 299\*299 patches 中心的64\*64区域的label。那么在level 7上对应的就是8\*8个pixel. 另外，testing时也切掉周边的10%。
* 思考一下，发现把tumor check region 从128@lev0改成128@lev3之后，取出的patches对应的center region变大了。如果training是用sliding而不是random sample,很可能都没有多少label 1的training data. 而为了balance class, 从一张slide上提取出的数据也会很少。这也支持了我们采用random sample而非sliding window 在training中。

**0. 整体思路**

先试level 4, 然后level2自己。然后结合level 2+level4 作为multi-scale。跟paper上保持一致。

**1. 图像裁剪问题**

在一切开始之前，看下老师提供的所有slides的样子。做出了每一个图level 7的图像。有些大片癌细胞，有些level7图像上不明显看到。只有91图是8个level, 其他的有9层，有10层

可以发现基本组织图像都在中间，癌细胞更是在中间。我们关心的应该是如何在组织中找到癌细胞。如果patches中组织的成分都很少，那就学习的意义不大。另外，我们判断tissue的方式是grayscale>0.8,可以发现在图像边缘的部分，可能因为载玻片边缘变形各种，也会误认为是组织。之前，因为癌细胞占整体比重小，如果随机sample, 会出现imblanced classes. 但如果不对图像进行裁剪，并非真实组织的图像会被采用与label 0 的training class. 所以对这一部分，直接不用于training.取横纵坐标10%-90%的也就是相当于原图64%的部分进行training.

那么对应的test. 这样不影响。因为我们是对高resolution的处理，从level7的10%-90%的pixel开始一个个predict,这样还不用对高resolution进行padding。实在不行就只给出patches的热力图，不用给整体。拟test用001，002， 101，110。前两个竖长图像且癌细胞少，后两个横、大图像且癌细胞多。

又注意，training的时候是选的tissue center在10%-90%范围内，我们关心的始终是center region，不关心边上299跑到哪。而有了10%的限制，299的一半150 < level5短边的1600+，我们不用担心会超出边缘。

**1.2.图像处理之二**

即使加了10%的裁剪和grayscale=0.8的过滤，发现产生的label 0的图像中还是很多灰的部分。反之，label1的图像就很紫。担心model会把这种灰紫差别（也就是是否tissue当作是否tumor的判断依据）。跟同学讨论了之后，发现都有这种情况。开始想的是，是不是对patches取判断，对以sampled tissue pixel长出的patch, 判断是否有一半以上属于tissue，是才留下来。然后Amal建议说可以改判断tissue的intensity，按paper给的是grayscale<0.8，试着改成了grayscale<0.5, 发现这样形成的图像从原本的占30%下降到了19%，并且生成的label 0 的确tissue的部分多了的样子。所以就这么样吧。

**把生成training图像写成了function, 默认的level是4，tissue intensity是0.5**

**1.3. All black in training images/23**

可能是数据格式问题？先不管了。23本来肿瘤也很少。生成要很久。算了。



**2. Test的处理**

I found the number 128\*128 center pixels (in level 0) to check whether tumor inside any patch is meaningful. I guess they choose 128\*128 (in level 0) because it is 1\*1 pixel in level 7. So whatever level you are using, your predicted region is one pixel in level 7. To make the predictions on level 7 continuous, we need to slide windows on the specific level we are training on. Take level 4 as example, 1\*1 pixel in level 7 corresponds to 8\*8 in level 4. So every input 299\*299 on level 4 gives prediction of center 8\*8 (~1\*1 in level 7 ~ 128\*128 in level 0). Then we stride with step 8 to give prediction of next 8\*8 in level 4 (so next pixel in level 7). There is no averaging here if we don't use data augmentation or multi-scale.

现在的问题是，如果level7每个pixel都要test，比如110slide的level 7 是736*560， 那就相当于350000个testing image, 即使我只删选出tissue area(又增加麻烦)，也还有将近1000000个。决定采用Amal的建议，选取更大的checking region.

决定取level3作为high resolution, level4作为lower resolution. 取level3上的128*128 作为checking tumor region, 这样level4上就是predict 299\*299 patches 中心的64\*64区域的label。那么在level 7上对应的就是8\*8个pixel. 另外，testing时也切掉周边的10%。

并且发现[`feature_extraction.image.extract_patches_2d`]可以取patches，但是不能有stride.所以自己loop 写。

所以步骤：

1. 读取level4, 只取10%开始。
2. 做nested loop，stride 64. 10%边长+64n恰好过一点点90%边长结束。两边都是这样。生成testing image 扔进drive.
3. 导进model predict.

**3. 对于multi-scale的思考**

fully connect: 再加一层dense layer，把每个小模型的输出concatenate当作input . 整体train，因为每个小模型本身就是pretrained.

考虑到每个通过logits再连的，应该是include_top=True?

看https://github.com/flyyufelix/cnn_finetune/blob/master/inception_v3.py 

也有merge的layer

https://stackoverflow.com/questions/51091553/how-to-merge-2-simillar-inception-models-with-identical-input-in-keras





