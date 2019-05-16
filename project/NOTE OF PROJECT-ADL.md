### NOTE OF PROJECT-ADL

**0. 整体思路**

先试level 4, 然后level2自己。然后结合level 2+level4 作为multi-scale。跟paper上保持一致。

**1. 图像裁剪问题**

在一切开始之前，看下老师提供的所有slides的样子。做出了每一个图level 7的图像。有些大片癌细胞，有些level7图像上不明显看到。

可以发现基本组织图像都在中间，癌细胞更是在中间。我们关心的应该是如何在组织中找到癌细胞。如果patches中组织的成分都很少，那就学习的意义不大。另外，我们判断tissue的方式是grayscale>0.8,可以发现在图像边缘的部分，可能因为载玻片边缘变形各种，也会误认为是组织。之前，因为癌细胞占整体比重小，如果随机sample, 会出现imblanced classes. 但如果不对图像进行裁剪，并非真实组织的图像会被采用与label 0 的training class. 所以对这一部分，直接不用于training.取横纵坐标10%-90%的也就是相当于原图64%的部分进行training.

那么对应的test. 这样不影响。因为我们是对高resolution的处理，从level7的10%-90%的pixel开始一个个predict,这样还不用对高resolution进行padding。实在不行就只给出patches的热力图，不用给整体。拟test用001，002， 101，110。前两个竖长图像且癌细胞少，后两个横、大图像且癌细胞多。

又注意，training的时候是选的tissue center在10%-90%范围内，我们关心的始终是center region，不关心边上299跑到哪。而有了10%的限制，299的一半150 < level5短边的1600+，我们不用担心会超出边缘。



**2. Test的处理**

I found the number 128\*128 center pixels (in level 0) to check whether tumor inside any patch is meaningful. I guess they choose 128\*128 (in level 0) because it is 1\*1 pixel in level 7. So whatever level you are using, your predicted region is one pixel in level 7. To make the predictions on level 7 continuous, we need to slide windows on the specific level we are training on. Take level 4 as example, 1\*1 pixel in level 7 corresponds to 8\*8 in level 4. So every input 299\*299 on level 4 gives prediction of center 8\*8 (~1\*1 in level 7 ~ 128\*128 in level 0). Then we stride with step 8 to give prediction of next 8\*8 in level 4 (so next pixel in level 7). There is no averaging here if we don't use data augmentation or multi-scale.

**3. 对于multi-scale的思考**



