# Quick Reference - Remote Control Raspberry Pi

## 🎮 Keyboard Controls

| Key | Action |
|-----|--------|
| `↑` | Increase UI scale (+0.05) |
| `↓` | Decrease UI scale (-0.05) |
| `F` | Toggle fullscreen |
| `H` | Emergency halt (stop all) |
| `R` | Scan serial ports |
| `D` | Toggle debug mode |
| `P` | Toggle rotator power |
| `1-6` | Select antenna 1-6 |
| `ESC` | Return to control screen |

## 📊 UI Scale

- **Default**: 0.6x (optimal for 800x480)
- **Range**: 0.4x to 1.5x
- **Step**: 0.05 per key press
- **Indicator**: Bottom-right corner shows current scale

## 📱 Display Support

- **Primary**: Raspberry Pi 7" (800x480)
- **Works on**: Any display with adjustable scaling
- **Touchscreen**: Fully supported
- **Mouse**: Fully supported

## 🔧 Quick Setup

```bash
# 1. Navigate to sketch folder
cd /path/to/RemoteControl_RaspberryPi

# 2. Run with Processing
processing-java --sketch=$(pwd) --run

# 3. Adjust scale with ↑↓ keys
```

## 📁 Files

- `RemoteControl_RaspberryPi.pde` - Main sketch (use this)
- `README.md` - Original code (reference only)
- `RASPBERRY_PI_README.md` - Full documentation
- `IMPLEMENTATION_SUMMARY.md` - Technical details

## ⚡ Features

✅ Raspberry Pi 7" optimized (800x480)  
✅ Adjustable UI scaling (0.4x - 1.5x)  
✅ Fullscreen mode  
✅ Touch/mouse precision at any scale  
✅ All original features maintained  
✅ Real-time scale indicator  

## 🎯 Usage Tips

1. **First Run**: Start with default 0.6x scale
2. **Too Small?**: Press `↑` to increase scale
3. **Too Large?**: Press `↓` to decrease scale
4. **Fullscreen**: Press `F` for maximum screen usage
5. **Check Indicator**: Bottom-right shows current scale

## 🐛 Troubleshooting

**Problem**: UI too small/large  
**Solution**: Use ↑↓ keys to adjust scale

**Problem**: Mouse clicks not registering  
**Solution**: All clicks should work - check if systemOn (top-right toggle)

**Problem**: Want bigger/smaller range  
**Solution**: Edit MIN_SCALE/MAX_SCALE in the code

## 📞 Support

See `RASPBERRY_PI_README.md` for complete documentation  
See `IMPLEMENTATION_SUMMARY.md` for technical details
