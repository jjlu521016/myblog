---
title: java 图片加水印（图片或者文本）
toc: true
date: 2017-05-11 13:58:13
tags:
    - java
    - 图片加水印
---
对于上传的图片，有时候我们需要加上水印来标识图片的来源，以下java代码用来处理图片加文字和图片水印
<!-- more -->
```java
import org.springframework.util.StringUtils;

import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.font.FontRenderContext;
import java.awt.geom.Rectangle2D;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
/**
 * @author jasonLu
 * @date 2017/5/11 12:30
 * @Description:图片添加水印 水印适用于 图片或文字
 */
public class WaterMaskImgUtils
{

    /**
     * 添加图片水印
     *
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param waterImg 水印图片路径，如：C://myPictrue//logo.png
     * @param x 水印图片距离目标图片左侧的偏移量，如果x<0, 则在正中间
     * @param y 水印图片距离目标图片上侧的偏移量，如果y<0, 则在正中间
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     * @param degree 水印图片旋转角度
     */
    public static void pressImage(String targetImg, String waterImg,  int x, int y, float alpha, Integer degree,String suffix)
    {
         pressImage(targetImg,waterImg,null,x,y,alpha,null,suffix);
    }

    /**
     * 添加图片水印
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param waterImg 水印图片路径，如：C://myPictrue//logo.png
     * @param outImg 图片输出位置，如果为空，则覆盖原文件
     * @param x 水印图片距离目标图片左侧的偏移量，如果x<0, 则在正中间
     * @param y 水印图片距离目标图片上侧的偏移量，如果y<0, 则在正中间
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     * @param degree 水印图片旋转角度
     */
    public static void pressImage(String targetImg, String waterImg, String outImg, int x, int y, float alpha, Integer degree,String suffix)
    {
        try
        {
            File file = new File(targetImg);
            //如果没有指定文件存放地址，则默认替换掉原图片
            File outFile;
            if(StringUtils.isEmpty(outImg))
            {
                outFile = file;
            }
            else
            {
                outFile = new File(outImg);
            }

            Image image = null;
            try
            {
                image = ImageIO.read(file);
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
            int width = image.getWidth(null);
            int height = image.getHeight(null);
            BufferedImage bufferedImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
            Graphics2D g = bufferedImage.createGraphics();
            g.drawImage(image, 0, 0, width, height, null);

            if (null != degree)
            {
                // 设置水印旋转
                g.rotate(Math.toRadians(degree), (double) bufferedImage.getWidth() / 2, (double) bufferedImage.getHeight() / 2);
            }

            Image waterImage = ImageIO.read(new File(waterImg));    // 水印文件
            int width_1 = waterImage.getWidth(null);
            int height_1 = waterImage.getHeight(null);
            g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_ATOP, alpha));

            int widthDiff = width - width_1;
            int heightDiff = height - height_1;
            if (x < 0)
            {
                x = widthDiff / 2;
            }
            else if (x > widthDiff)
            {
                x = widthDiff;
            }
            if (y < 0)
            {
                y = heightDiff / 2;
            }
            else if (y > heightDiff)
            {
                y = heightDiff;
            }
            g.drawImage(waterImage, x, y, width_1, height_1, null); // 水印文件结束
            g.dispose();
            //注意这里的后缀不能带 .
            ImageIO.write(bufferedImage, suffix.substring(1), outFile);
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }

    }

    /**
     * 添加文字水印
     *
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param pressText 水印文字， 如：云账房
     * @param fontName 字体名称，    如：宋体
     * @param fontStyle 字体样式，如：粗体和斜体(Font.BOLD|Font.ITALIC)
     * @param fontSize 字体大小，单位为像素
     * @param color 字体颜色
     * @param x 水印文字距离目标图片左侧的偏移量，如果x<0, 则在正中间
     * @param y 水印文字距离目标图片上侧的偏移量，如果y<0, 则在正中间
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     */
    public static void pressText(String targetImg, String pressText, String fontName, int fontStyle, int fontSize, Color color, int x, int y, float alpha,String suffix)
    {
        pressText(targetImg,pressText,null,fontName,fontStyle,fontSize,color,x,y,alpha,null,suffix);
    }

    /**
     * 添加文字水印
     * @Description 坐标(0,0)在图片的左上角，(负数，负数)在中心位置，(0,xxx)在左边，(x,0)在最上边，其他位置根据图片来确定
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param pressText 水印文字， 如：云账房
     * @param outImg 图片输出位置，如果为空，则覆盖原文件
     * @param fontName 字体名称，    如：宋体
     * @param fontStyle 字体样式，如：粗体和斜体(Font.BOLD|Font.ITALIC)
     * @param fontSize 字体大小，单位为像素
     * @param color 字体颜色
     * @param positionX 水印文字距离目标图片左侧的偏移量，如果x<0, 则在正中间
     * @param positionY 水印文字距离目标图片上侧的偏移量，如果y<0, 则在正中间
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     * @param degree 水印图片旋转角度
     */
    public static void pressText(String targetImg, String pressText, String outImg, String fontName, int fontStyle, int fontSize, Color color, int positionX, int positionY, float alpha, Integer degree,String suffix)
    {
        try
        {
            File file = new File(targetImg);
            // 如果没有指定文件存放地址，则默认替换掉原图片
            File outFile;
            if (StringUtils.isEmpty(outImg))
            {
                outFile = file;
            }
            else
            {
                outFile = new File(outImg);
            }
            Image image = ImageIO.read(file);
            int width = image.getWidth(null);
            int height = image.getHeight(null);
            BufferedImage bufferedImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
            Font font = new Font(fontName, fontStyle, fontSize);
            Graphics2D g = bufferedImage.createGraphics();
            g.drawImage(image, 0, 0, width, height, null);
            g.setFont(font);
            g.setColor(color);
            g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_ATOP, alpha));
            if (null != degree)
            {
                // 设置水印旋转
                g.rotate(Math.toRadians(degree), (double) bufferedImage.getWidth() / 2, (double) bufferedImage.getHeight() / 2);
            }
            //获取文字所占的像素
            FontRenderContext context = g.getFontRenderContext();
            Rectangle2D stringBounds = font.getStringBounds(pressText,context);

            int textWidth = (int) stringBounds.getWidth() ;
            int textHeight = (int) stringBounds.getHeight();

            int widthDiff = width - textWidth;
            int heightDiff = height - textHeight;
            if (positionX < 0)
            {
                positionX = widthDiff / 2;
            }
            else if (positionX > widthDiff)
            {
                positionX = widthDiff;
            }
            if (positionY < 0)
            {
                positionY = heightDiff / 2;
            }
            else if (positionY > heightDiff)
            {
                positionY = heightDiff;
            }

            g.drawString(pressText, positionX, positionY + textHeight);
            g.dispose();
            ImageIO.write(bufferedImage, suffix, outFile);

        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }


    /**
     * 添加文字水印
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param pressText 水印文字， 如：云账房
     * @param fontName 字体名称，    如：宋体
     * @param fontStyle 字体样式，如：粗体和斜体(Font.BOLD|Font.ITALIC)
     * @param fontSize 字体大小，单位为像素
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     */
    public static void pressTextToRightBottom(String targetImg, String pressText, String fontName, int fontStyle, int fontSize, Color color, float alpha,String suffix)
    {

        pressTextToRightBottom(targetImg,pressText,null,fontName,fontStyle,fontSize,color,alpha,null,suffix);
    }
    /**
     * 在图片的右下角添加水印
     *
     * @param targetImg 目标图片路径，如：C://myPictrue//1.jpg
     * @param pressText 水印文字， 如：云账房
     * @param outImg 图片输出位置，如果为空，则覆盖原文件
     * @param fontName 体名称， 如：宋体
     * @param fontStyle 字体样式，如：粗体和斜体(Font.BOLD|Font.ITALIC)
     * @param fontSize 字体大小，单位为像素
     * @param alpha 透明度(0.0 -- 1.0, 0.0为完全透明，1.0为完全不透明)
     * @param degree 水印图片旋转角度
     */
    public static void pressTextToRightBottom(String targetImg, String pressText, String outImg, String fontName, int fontStyle, int fontSize, Color color, float alpha, Integer degree,String suffix)
    {
        try
        {
            File file = new File(targetImg);
            File outFile;
            if (StringUtils.isEmpty(outImg))
            {
                outFile = file;
            }
            else
            {
                outFile = new File(outImg);
            }
            Image image = ImageIO.read(file);
            int width = image.getWidth(null);
            int height = image.getHeight(null);
            BufferedImage bufferedImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
            Font font = new Font(fontName, fontStyle, fontSize);
            Graphics2D g = bufferedImage.createGraphics();
            g.drawImage(image, 0, 0, width, height, null);
            g.setFont(font);
            g.setColor(color);
            g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_ATOP, alpha));
            if (null != degree)
            {
                // 设置水印旋转
                g.rotate(Math.toRadians(degree), (double) bufferedImage.getWidth() / 2, (double) bufferedImage.getHeight() / 2);
            }

            //获取文字所占的像素
            FontRenderContext context = g.getFontRenderContext();
            Rectangle2D stringBounds = font.getStringBounds(pressText,context);

            int textWidth = (int) stringBounds.getWidth() ;
            int textHight = (int) stringBounds.getHeight();

            g.drawString(pressText, width - textWidth, height - textHight);
            g.dispose();
            ImageIO.write(bufferedImage, suffix.substring(1), outFile);
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }
}
```