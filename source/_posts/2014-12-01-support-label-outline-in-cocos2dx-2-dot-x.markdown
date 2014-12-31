---
layout: post
title: "让Cocos2d-x 2.x版本CCLabelTTF支持外描边(非多个方向重绘来实现描边)"
date: 2014-12-01 17:12:58 +0800
comments: true
categories: 
---

想看实现方式请直接跳到Part 2。

Part 1:
我上一个游戏项目使用的是Cocos2d-x 2.1.2，那时候CCLabelTTF还没有支持描边的API，后来的版本加上了enableStroke这个接口，可是实现起来的效果是内描边，内描边的效果并不是很理想，如果字体本身很细的话，内描边后，基本全是描边，没剩多少肉了。

当时搜寻网上，找到一个可行的外描边方案。这个方案是修改CCLabelTTF::updateTexture()这个方法，原理是将原来字体的纹理，往字的360度方向位移描边的宽度距离，然后绘制生成新的纹理，如果每40度绘制一次的话，360度就需要绘制9次，最后再将字体本身绘制一次，本来一个label 的drawcall消耗是1次，这种方案drawcall消耗就是10次。

这个方案能够勉强实现外描边的效果，在苹果设备上的效果还是可以的，但是一拿到android设备上检查，就会发现描边并不是连贯的，有点坑坑洼洼的感觉。更让人无法接受的是，该方案会导致label的draw call剧烈增加，一些label比较多的页面，如果都需要描边效果的话，会导致游戏每一帧的绘制时间变长，造成游戏明显的卡顿。这个问题在android的低端设备上尤其明显。于是项目后期，我们放弃了字体描边的效果。

下面为该方案的源码：

<!--more-->

{% codeblock lang:c++ %}
//CCLabelTTF::updateTexture()
CCTexture2D *tex;
    
// let system compute label's width or height when its value is 0
// refer to cocos2d-x issue #1430
tex = new CCTexture2D();
if (!tex)
{
	return false;
}

tex->initWithString( m_string.c_str(),
                    m_pFontName->c_str(),
                    m_fFontSize * CC_CONTENT_SCALE_FACTOR(),
                    CC_SIZE_POINTS_TO_PIXELS(m_tDimensions),
                    m_hAlignment,
                    m_vAlignment);

tex->autorelease();

if (m_strokeEnabled) {
    
    //原始字符串
    CCSprite* spriteText = CCSprite::createWithTexture(tex);
    ccBlendFunc originalBlend = spriteText->getBlendFunc();
    
    //计算考虑描边后texture的尺寸
    CCSize realTextureSize = spriteText->getContentSize();
    realTextureSize.width += 2 * m_strokeSize;
    realTextureSize.height += 2 * m_strokeSize;
    
    //call to clear error
    glGetError();
    
    //创建用于实现描边的可渲染texture
    CCRenderTexture *rt = CCRenderTexture::create(realTextureSize.width, realTextureSize.height);
    
    //绘制底纹
    spriteText->setColor(m_strokeColor);
    ccBlendFunc func = { GL_SRC_ALPHA, GL_ONE};
    spriteText->setBlendFunc(func);
    spriteText->setAnchorPoint(CCPoint(0.5f, 0.5f));
    spriteText->setFlipY(true);
    
    int precision = 40; //此值越小，描边越细腻，同时CPU开销越高
    rt->begin();
    for(int i = 0; i < 360; i += precision)
    {
        float r = CC_DEGREES_TO_RADIANS(i);
        spriteText->setPosition(CCPoint(
                                        realTextureSize.width * 0.5f + sin(r) * m_strokeSize,
                                        realTextureSize.height * 0.5f + cos(r) * m_strokeSize));
        spriteText->visit();
    }
    
    //绘制原始字符串
    spriteText->setColor(this->getColor());
    spriteText->setBlendFunc(originalBlend);
    spriteText->setPosition(CCPoint(realTextureSize.width * 0.5f, realTextureSize.height * 0.5f));
    spriteText->visit();
    rt->end();
    
    tex = rt->getSprite()->getTexture();
    tex->setAliasTexParameters();
    
}

{% endcodeblock %}

Part 2：
目前这个项目因为开始的比较早，依然使用的是Cocos2d-x 2.x版本，虽然知道Cocos2d-x 3.x版本已经支持外描边，但是因为3.x相对于2.x改动很大，不太适合直接将项目引擎直接从2.x升级到3.x版本。既然没办法直接用上3.x的特性，那只好看看能不能借鉴借鉴3.x CCLabel的实现。

3.x中CCLabel的实现使用了新的方式，不像2.x中先通过生成纹理，再显示，这个纹理的生成依赖于平台各自的API，不同的平台各有实现方式，不过在3.x中，CCLabel的显示是直接由特定的着色器提供支持，要想让2.x的CCLabelTTF改用类似的方案也是可行的，只不过要费时费力一些。

幸运的是，3.x为了兼容以前CCLabelTTF的API，也提供了原来的实现，即生成纹理，然后draw，顺带将内描边改成了外描边，只不过触控目前好像已经不对2.x版本提供支持，并没有将对内描边的修正应用到2.x版本上。

3.x 外描边的修正可以参考下面这个链接：
https://github.com/Dhilan007/cocos2d-x/commit/6f9d379beb6f47f43a3305b885000d44e978ccd7

可以看到，这个修正主要改动了两个文件，一个是Cocos2dxBitmap.java中的createTextBitmapShadowStroke，另一个是CCDevice.mm的_initWithString。

第一个文件是Android平台上生成纹理的代码，在2.2.3里面，文件名和方法名也是一样的。下面是基于2.2.3修改的代码。

{% codeblock lang:java %}
public static void createTextBitmapShadowStroke(String pString,  final String pFontName, final int pFontSize,
													final float fontTintR, final float fontTintG, final float fontTintB,
													final int pAlignment, final int pWidth, final int pHeight, final boolean shadow,
													final float shadowDX, final float shadowDY, final float shadowBlur, final boolean stroke,
													final float strokeR, final float strokeG, final float strokeB, final float strokeSize) {
		
	final int horizontalAlignment = pAlignment & 0x0F;
	final int verticalAlignment   = (pAlignment >> 4) & 0x0F;

	pString = Cocos2dxBitmap.refactorString(pString);
	final Paint paint = Cocos2dxBitmap.newPaint(pFontName, pFontSize, horizontalAlignment);
	
	/**
	 * if the first word width less than designed width,It means no words to show
	 */
	if(0 != pWidth)
	{
		final int firstWordWidth = (int) Math.ceil(paint.measureText(pString, 0,1));
		if ( firstWordWidth > pWidth)
		{
			Log.w("createTextBitmapShadowStroke warning:","the input width is less than the width of the pString's first word\n");
			return;
		}
	}

	// set the paint color
	paint.setARGB(255, (int)(255.0 * fontTintR), (int)(255.0 * fontTintG), (int)(255.0 * fontTintB));

	final TextProperty textProperty = Cocos2dxBitmap.computeTextProperty(pString, pWidth, pHeight, paint);
	final int bitmapTotalHeight = (pHeight == 0 ? textProperty.mTotalHeight: pHeight);
	
	// padding needed when using shadows (not used otherwise)
	float bitmapPaddingX   = 0.0f;
	float bitmapPaddingY   = 0.0f;
	float renderTextDeltaX = 0.0f;
	float renderTextDeltaY = 0.0f;
	
	if (0 == textProperty.mMaxWidth || 0 == bitmapTotalHeight)
	{
		Log.w("createTextBitmapShadowStroke warning:","textProperty MaxWidth is 0 or bitMapTotalHeight is 0\n");
		return;
	}

	final Bitmap bitmap = Bitmap.createBitmap(textProperty.mMaxWidth + (int)bitmapPaddingX,
			bitmapTotalHeight + (int)bitmapPaddingY, Bitmap.Config.ARGB_8888);
	
	final Canvas canvas = new Canvas(bitmap);

	/* Draw string. */
	final FontMetricsInt fontMetricsInt = paint.getFontMetricsInt();
	
	// draw again with stroke on if needed 
	if ( stroke ) {
		
		final Paint paintStroke = Cocos2dxBitmap.newPaint(pFontName, pFontSize, horizontalAlignment);
		paintStroke.setStyle(Paint.Style.STROKE);
		paintStroke.setStrokeWidth(strokeSize);
		paintStroke.setARGB(255, (int)strokeR * 255, (int)strokeG * 255, (int)strokeB * 255);
		
		int x = 0;
		int y = Cocos2dxBitmap.computeY(fontMetricsInt, pHeight, textProperty.mTotalHeight, verticalAlignment);
		final String[] lines2 = textProperty.mLines;
		
		for (final String line : lines2) {
			
			x = Cocos2dxBitmap.computeX(line, textProperty.mMaxWidth, horizontalAlignment);
			canvas.drawText(line, x + renderTextDeltaX, y + renderTextDeltaY, paintStroke);
			canvas.drawText(line, x + renderTextDeltaX, y + renderTextDeltaY, paint);
			y += textProperty.mHeightPerLine;
			
		}
		
	}
	else
	{
		int x = 0;
		int y = Cocos2dxBitmap.computeY(fontMetricsInt, pHeight, textProperty.mTotalHeight, verticalAlignment);
		
		final String[] lines = textProperty.mLines;
		
		for (final String line : lines) {
			
			x = Cocos2dxBitmap.computeX(line, textProperty.mMaxWidth, horizontalAlignment);
			canvas.drawText(line, x + renderTextDeltaX, y + renderTextDeltaY, paint);
			y += textProperty.mHeightPerLine;
			
		}
	}
	
	Cocos2dxBitmap.initNativeObject(bitmap);
}

{% endcodeblock %}

第二个文件是iOS平台生成纹理的代码。在2.2.3中相应的代码在CCImage.mm中，方法名还是一样的。下面是基于2.2.3修改的代码。

{% codeblock lang:objective-c %}

static bool s_isIOS7OrHigher = false;

static inline void lazyCheckIOS7()
{
    static bool isInited = false;
    if (!isInited)
    {
        s_isIOS7OrHigher = [[[UIDevice currentDevice] systemVersion] compare:@"7.0" options:NSNumericSearch] != NSOrderedAscending;
        isInited = true;
    }
}

// refer CCImage::ETextAlign
#define ALIGN_TOP    1
#define ALIGN_CENTER 3
#define ALIGN_BOTTOM 2

static bool _initWithString(const char * pText, cocos2d::CCImage::ETextAlign eAlign, const char * pFontName, int nSize, tImageInfo* pInfo)
{
    // lazy check whether it is iOS7 device
    lazyCheckIOS7();
    
    bool bRet = false;
    do 
    {
        CC_BREAK_IF(! pText || ! pInfo);
        
        NSString * str          = [NSString stringWithUTF8String:pText];
        NSString * fntName      = [NSString stringWithUTF8String:pFontName];
        
        CGSize dim, constrainSize;
        
        constrainSize.width     = pInfo->width;
        constrainSize.height    = pInfo->height;
        
        // On iOS custom fonts must be listed beforehand in the App info.plist (in order to be usable) and referenced only the by the font family name itself when
        // calling [UIFont fontWithName]. Therefore even if the developer adds 'SomeFont.ttf' or 'fonts/SomeFont.ttf' to the App .plist, the font must
        // be referenced as 'SomeFont' when calling [UIFont fontWithName]. Hence we strip out the folder path components and the extension here in order to get just
        // the font family name itself. This stripping step is required especially for references to user fonts stored in CCB files; CCB files appear to store
        // the '.ttf' extensions when referring to custom fonts.
        fntName = [[fntName lastPathComponent] stringByDeletingPathExtension];
        
        // create the font   
        id font = [UIFont fontWithName:fntName size:nSize];
        
        if (font)
        {
            dim = _calculateStringSize(str, font, &constrainSize);
        }
        else
        {
            if (!font)
            {
                font = [UIFont systemFontOfSize:nSize];
            }
                
            if (font)
            {
                dim = _calculateStringSize(str, font, &constrainSize);
            }
        }

        CC_BREAK_IF(! font);
        
        // compute start point
        int startH = 0;
        if (constrainSize.height > dim.height)
        {
            // vertical alignment
            unsigned int vAlignment = (eAlign >> 4) & 0x0F;
            if (vAlignment == ALIGN_TOP)
            {
                startH = 0;
            }
            else if (vAlignment == ALIGN_CENTER)
            {
                startH = (constrainSize.height - dim.height) / 2;
            }
            else 
            {
                startH = constrainSize.height - dim.height;
            }
        }
        
        // adjust text rect
        if (constrainSize.width > 0 && constrainSize.width > dim.width)
        {
            dim.width = constrainSize.width;
        }
        if (constrainSize.height > 0 && constrainSize.height > dim.height)
        {
            dim.height = constrainSize.height;
        }
        
        
        // compute the padding needed by shadow and stroke
        float shadowStrokePaddingX = 0.0f;
        float shadowStrokePaddingY = 0.0f;
        
        if ( pInfo->hasStroke )
        {
            shadowStrokePaddingX = ceilf(pInfo->strokeSize);
            shadowStrokePaddingY = ceilf(pInfo->strokeSize);
        }
        
        if ( pInfo->hasShadow )
        {
            shadowStrokePaddingX = std::max(shadowStrokePaddingX, (float)abs(pInfo->shadowOffset.width));
            shadowStrokePaddingY = std::max(shadowStrokePaddingY, (float)abs(pInfo->shadowOffset.height));
        }
        
        // add the padding (this could be 0 if no shadow and no stroke)
        dim.width  += shadowStrokePaddingX*2;
        dim.height += shadowStrokePaddingY*2;
        
        
        unsigned char* data = new unsigned char[(int)(dim.width * dim.height * 4)];
        memset(data, 0, (int)(dim.width * dim.height * 4));
        
        // draw text
        CGColorSpaceRef colorSpace  = CGColorSpaceCreateDeviceRGB();
        CGContextRef context        = CGBitmapContextCreate(data,
                                                            dim.width,
                                                            dim.height,
                                                            8,
                                                            (int)(dim.width) * 4,
                                                            colorSpace,
                                                            kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big);
        
        CGColorSpaceRelease(colorSpace);
        
        if (!context)
        {
            delete[] data;
            break;
        }

        // text color
        CGContextSetRGBFillColor(context, pInfo->tintColorR, pInfo->tintColorG, pInfo->tintColorB, 1);
        // move Y rendering to the top of the image
        CGContextTranslateCTM(context, 0.0f, (dim.height - shadowStrokePaddingY) );
        CGContextScaleCTM(context, 1.0f, -1.0f); //NOTE: NSString draws in UIKit referential i.e. renders upside-down compared to CGBitmapContext referential
        
        // store the current context
        UIGraphicsPushContext(context);
        
        // measure text size with specified font and determine the rectangle to draw text in
        unsigned uHoriFlag = eAlign & 0x0f;
        UITextAlignment align = (UITextAlignment)((2 == uHoriFlag) ? UITextAlignmentRight
                                : (3 == uHoriFlag) ? UITextAlignmentCenter
                                : UITextAlignmentLeft);
        
        //------------------------------------------------------------------------------------
        
        // compute the rect used for rendering the text
        // based on wether shadows or stroke are enabled
        
        float textOriginX  = 0;
        float textOrigingY = startH;
        
        float textWidth    = dim.width;
        float textHeight   = dim.height;
        
        CGRect rect = CGRectMake(textOriginX, textOrigingY, textWidth, textHeight);
        
        CGContextSetShouldSubpixelQuantizeFonts(context, false);
        
        CGContextBeginTransparencyLayerWithRect(context, rect, NULL);
        
        if ( pInfo->hasStroke )
        {
            CGContextSetTextDrawingMode(context, kCGTextStroke);
            
            if(s_isIOS7OrHigher)
            {
                NSMutableParagraphStyle *paragraphStyle = [[NSMutableParagraphStyle alloc] init];
                paragraphStyle.alignment = align;
                paragraphStyle.lineBreakMode = NSLineBreakByWordWrapping;
                [str drawInRect:rect withAttributes:@{
                                                      NSFontAttributeName: font,
                                                      NSStrokeWidthAttributeName: [NSNumber numberWithFloat: pInfo->strokeSize / nSize * 100 ],
                                                      NSForegroundColorAttributeName:[UIColor colorWithRed:pInfo->tintColorR
                                                                                                     green:pInfo->tintColorG
                                                                                                      blue:pInfo->tintColorB
                                                                                                     alpha:1.0f],
                                                      NSParagraphStyleAttributeName:paragraphStyle,
                                                      NSStrokeColorAttributeName: [UIColor colorWithRed:pInfo->strokeColorR
                                                                                                  green:pInfo->strokeColorG
                                                                                                   blue:pInfo->strokeColorB
                                                                                                  alpha:1.0f]
                                                      }
                 ];
                
                [paragraphStyle release];
            }
            else
            {
                CGContextSetRGBStrokeColor(context, pInfo->strokeColorR, pInfo->strokeColorG, pInfo->strokeColorB, 1);
                CGContextSetLineWidth(context, pInfo->strokeSize);
                
                //original code that was not working in iOS 7
                [str drawInRect: rect withFont:font lineBreakMode:NSLineBreakByWordWrapping alignment:align];
            }
        }
        
        CGContextSetTextDrawingMode(context, kCGTextFill);
        
        // actually draw the text in the context
        [str drawInRect: rect withFont:font lineBreakMode:NSLineBreakByWordWrapping alignment:align];
        
        CGContextEndTransparencyLayer(context);
        
        // pop the context
        UIGraphicsPopContext();
        
        // release the context
        CGContextRelease(context);
               
        // output params
        pInfo->data                 = data;
        pInfo->hasAlpha             = true;
        pInfo->isPremultipliedAlpha = true;
        pInfo->bitsPerComponent     = 8;
        pInfo->width                = dim.width;
        pInfo->height               = dim.height;
        bRet                        = true;
        
    } while (0);

    return bRet;
}

{% endcodeblock %}

修改完上述两个文件，Cococs2d-x 2.x版本就能完美支持外描边啦。