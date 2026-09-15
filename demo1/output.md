# Image Resizer Script

Here's a comprehensive Python script to resize images with multiple options:

## Basic Version

```python
#!/usr/bin/env python3
"""
Simple image resizer script
Supports common image formats: JPG, PNG, GIF, BMP, WEBP
"""

from PIL import Image
import os
import sys

def resize_image(input_path, output_path, width=None, height=None, maintain_aspect=True):
    """
    Resize an image to specified dimensions
    
    Args:
        input_path: Path to input image
        output_path: Path to save resized image
        width: Target width (None to auto-calculate)
        height: Target height (None to auto-calculate)
        maintain_aspect: Keep aspect ratio
    """
    try:
        # Open image
        img = Image.open(input_path)
        original_width, original_height = img.size
        
        # Calculate dimensions
        if maintain_aspect:
            if width and not height:
                height = int(original_height * (width / original_width))
            elif height and not width:
                width = int(original_width * (height / original_height))
            elif width and height:
                # Fit within bounds while maintaining aspect ratio
                ratio = min(width / original_width, height / original_height)
                width = int(original_width * ratio)
                height = int(original_height * ratio)
        
        # Resize image
        resized_img = img.resize((width, height), Image.Resampling.LANCZOS)
        
        # Save image
        resized_img.save(output_path, quality=95, optimize=True)
        print(f"✓ Resized: {input_path} -> {output_path} ({width}x{height})")
        
    except Exception as e:
        print(f"✗ Error processing {input_path}: {str(e)}")

# Example usage
if __name__ == "__main__":
    resize_image("input.jpg", "output.jpg", width=800)
```

## Advanced Version with CLI

```python
#!/usr/bin/env python3
"""
Advanced Image Resizer with Command Line Interface
Supports batch processing, multiple resize modes, and various formats
"""

from PIL import Image
import argparse
import os
import glob
from pathlib import Path

class ImageResizer:
    SUPPORTED_FORMATS = {'.jpg', '.jpeg', '.png', '.gif', '.bmp', '.webp', '.tiff'}
    
    def __init__(self, quality=95, optimize=True):
        self.quality = quality
        self.optimize = optimize
    
    def resize_fit(self, img, width, height):
        """Resize to fit within dimensions, maintaining aspect ratio"""
        img.thumbnail((width, height), Image.Resampling.LANCZOS)
        return img
    
    def resize_fill(self, img, width, height):
        """Resize and crop to fill dimensions exactly"""
        img_ratio = img.width / img.height
        target_ratio = width / height
        
        if img_ratio > target_ratio:
            # Image is wider, crop width
            new_width = int(img.height * target_ratio)
            offset = (img.width - new_width) // 2
            img = img.crop((offset, 0, offset + new_width, img.height))
        else:
            # Image is taller, crop height
            new_height = int(img.width / target_ratio)
            offset = (img.height - new_height) // 2
            img = img.crop((0, offset, img.width, offset + new_height))
        
        return img.resize((width, height), Image.Resampling.LANCZOS)
    
    def resize_stretch(self, img, width, height):
        """Resize without maintaining aspect ratio"""
        return img.resize((width, height), Image.Resampling.LANCZOS)
    
    def resize_percentage(self, img, percentage):
        """Resize by percentage"""
        width = int(img.width * percentage / 100)
        height = int(img.height * percentage / 100)
        return img.resize((width, height), Image.Resampling.LANCZOS)
    
    def process_image(self, input_path, output_path, width=None, height=None, 
                     mode='fit', percentage=None):
        """Process a single image"""
        try:
            with Image.open(input_path) as img:
                # Convert RGBA to RGB if saving as JPEG
                if img.mode == 'RGBA' and output_path.lower().endswith(('.jpg', '.jpeg')):
                    rgb_img = Image.new('RGB', img.size, (255, 255, 255))
                    rgb_img.paste(img, mask=img.split()[3])
                    img = rgb_img
                
                # Resize based on mode
                if percentage:
                    resized = self.resize_percentage(img, percentage)
                elif mode == 'fit':
                    resized = self.resize_fit(img.copy(), width, height)
                elif mode == 'fill':
                    resized = self.resize_fill(img.copy(), width, height)
                elif mode == 'stretch':
                    resized = self.resize_stretch(img, width, height)
                else:
                    raise ValueError(f"Unknown mode: {mode}")
                
                # Save image
                save_kwargs = {'optimize': self.optimize}
                if output_path.lower().endswith(('.jpg', '.jpeg')):
                    save_kwargs['quality'] = self.quality
                
                resized.save(output_path, **save_kwargs)
                
                return True, f"✓ {input_path} -> {output_path} ({resized.width}x{resized.height})"
        
        except Exception as e:
            return False, f"✗ Error: {input_path} - {str(e)}"
    
    def batch_process(self, input_pattern, output_dir, width=None, height=None, 
                     mode='fit', percentage=None, prefix='', suffix='_resized'):
        """Process multiple images"""
        files = []
        
        # Handle different input patterns
        if os.path.isdir(input_pattern):
            for ext in self.SUPPORTED_FORMATS:
                files.extend(glob.glob(os.path.join(input_pattern, f'*{ext}')))
                files.extend(glob.glob(os.path.join(input_pattern, f'*{ext.upper()}')))
        else:
            files = glob.glob(input_pattern)
        
        if not files:
            print(f"No images found matching: {input_pattern}")
            return
        
        # Create output directory
        os.makedirs(output_dir, exist_ok=True)
        
        # Process each file
        results = {'success': 0, 'failed': 0}
        for file_path in files:
            filename = Path(file_path).stem
            extension = Path(file_path).suffix
            output_filename = f"{prefix}{filename}{suffix}{extension}"
            output_path = os.path.join(output_dir, output_filename)
            
            success, message = self.process_image(
                file_path, output_path, width, height, mode, percentage
            )
            
            print(message)
            if success:
                results['success'] += 1
            else:
                results['failed'] += 1
        
        print(f"\nProcessed: {results['success']} successful, {results['failed']} failed")

def main():
    parser = argparse.ArgumentParser(
        description='Resize images with various options',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  # Resize single image to 800px width
  %(prog)s input.jpg -o output.jpg -w 800
  
  # Resize to fit within 800x600
  %(prog)s input.jpg -o output.jpg -w 800 -h 600 --mode fit
  
  # Batch resize all JPGs in folder
  %(prog)s "images/*.jpg" -od resized/ -w 1024
  
  # Resize to 50%% of original
  %(prog)s input.jpg -o output.jpg -p 50
  
  # Resize and crop to exact dimensions
  %(prog)s input.jpg -o output.jpg -w 800 -h 600 --mode fill
        """
    )
    
    parser.add_argument('input', help='Input image path or pattern (e.g., *.jpg)')
    parser.add_argument('-o', '--output', help='Output image path (for single image)')
    parser.add_argument('-od', '--output-dir', default='resized', 
                       help='Output directory (for batch processing)')
    parser.add_argument('-w', '--width', type=int, help='Target width in pixels')
    parser.add_argument('-h', '--height', type=int, help='Target height in pixels')
    parser.add_argument('-p', '--percentage', type=float, 
                       help='Resize by percentage (e.g., 50 for 50%%)')
    parser.add_argument('-m', '--mode', choices=['fit', 'fill', 'stretch'], 
                       default='fit', help='Resize mode (default: fit)')
    parser.add_argument('-q', '--quality', type=int, default=95, 
                       help='JPEG quality 1-100 (default: 95)')
    parser.add_argument('--prefix', default='', help='Output filename prefix')
    parser.add_argument('--suffix', default='_resized', help='Output filename suffix')
    parser.add_argument('--no-optimize', action='store_true', 
                       help='Disable image optimization')
    
    args = parser.parse_args()
    
    # Validation
    if not args.percentage and not (args.width or args.height):
        parser.error("Must specify --width, --height, or --percentage")
    
    # Create resizer
    resizer = ImageResizer(quality=args.quality, optimize=not args.no_optimize)
    
    # Check if single file or batch
    if os.path.isfile(args.input):
        # Single file processing
        if not args.output:
            parser.error("--output required for single file")
        
        success, message = resizer.process_image(
            args.input, args.output, args.width, args.height, 
            args.mode, args.percentage
        )
        print(message)
    else:
        # Batch processing
        resizer.batch_process(
            args.input, args.output_dir, args.width, args.height,
            args.mode, args.percentage, args.prefix, args.suffix
        )

if __name__ == "__main__":
    main()
```

## Installation

```bash
# Install required package
pip install Pillow

# Make script executable (Linux/Mac)
chmod +x resize_images.py
```

## Usage Examples

```bash
# Resize single image to 800px width
python resize_images.py input.jpg -o output.jpg -w 800

# Resize maintaining aspect ratio to fit in 1920x1080
python resize_images.py input.jpg -o output.jpg -w 1920 -h 1080

# Resize to 50% of original size
python resize_images.py input.jpg -o output.jpg -p 50

# Batch resize all JPGs in a folder
python resize_images.py "photos/*.jpg" -od resized/ -w 1024

# Resize and crop to exact dimensions (fill mode)
python resize_images.py input.jpg -o output.jpg -w 800 -h 600 -m fill

# Batch process with custom suffix
python resize_images.py images/ -od thumbnails/ -w 300 -h 300 --suffix _thumb
```

Choose the version that best fits your needs! The basic version is simpler, while the advanced version offers batch processing and multiple resize modes.
