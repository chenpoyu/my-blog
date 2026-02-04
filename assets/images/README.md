# 圖片資源說明

這個目錄存放部落格所需的圖片資源。

## 需要準備的圖片

### 1. Favicon 圖示
- `favicon-32x32.png` - 32x32 像素的網站圖示
- `favicon-16x16.png` - 16x16 像素的網站圖示
- `apple-touch-icon.png` - 180x180 像素的 Apple 裝置圖示

### 2. Logo
- `logo.png` - 網站 Logo，建議尺寸 200x200 像素

### 3. Open Graph 圖片
- `og-image.png` - 社群分享預覽圖，建議尺寸 1200x630 像素
- `default-post.png` - 文章預設分享圖，建議尺寸 1200x630 像素

## 圖片優化建議

1. 使用 WebP 格式以獲得更好的壓縮率
2. 圖片檔案大小應控制在 200KB 以內
3. 使用描述性的檔案名稱
4. 提供 alt 文字以改善 SEO 和無障礙性

## 使用 ImageMagick 壓縮圖片

```bash
# 壓縮並調整圖片大小
convert input.jpg -resize 1200x -quality 85 output.jpg

# 批次處理
for file in *.jpg; do
    convert "$file" -resize 1200x -quality 85 "optimized-$file"
done
```

## 製作 Favicon

```bash
# 從 PNG 製作 ICO
convert favicon.png -define icon:auto-resize=16,32,48,64 favicon.ico

# 製作不同尺寸的 favicon
convert logo.png -resize 32x32 favicon-32x32.png
convert logo.png -resize 16x16 favicon-16x16.png
convert logo.png -resize 180x180 apple-touch-icon.png
```
