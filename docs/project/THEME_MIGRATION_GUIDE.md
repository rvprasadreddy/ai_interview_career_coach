# Anti-Gravity Brand Kit - Flutter Migration Guide

## 📋 Table of Contents
1. [Overview](#overview)
2. [Setup Instructions](#setup-instructions)
3. [Understanding the Design System](#understanding-the-design-system)
4. [Step-by-Step Migration](#step-by-step-migration)
5. [Component Mapping](#component-mapping)
6. [Common Patterns](#common-patterns)
7. [Performance Optimization](#performance-optimization)
8. [Tips & Best Practices](#tips--best-practices)

---

## 🎯 Overview

This guide helps you apply the iOS-inspired Anti-Gravity design system to your existing Flutter app. The design system features:

- ✨ Floating UI elements with depth and elevation
- 🎨 Ethereal color palette (blues, purples, holographic accents)
- 🌊 Smooth physics-based animations
- 📱 iOS-style components and patterns
- 🌗 Light/Dark mode support
- ♿ Accessibility compliant (WCAG AA)

---

## 🚀 Setup Instructions

### 1. Add the Theme Files to Your Project

```
your_app/
  lib/
    theme/
      antigravity_theme.dart     ← Copy this file
    screens/
      home_screen.dart           ← Update your screens
    main.dart                    ← Update MaterialApp
```

### 2. Update pubspec.yaml

```yaml
dependencies:
  flutter:
    sdk: flutter
  # No additional packages required for basic implementation!
  # Optional: Add these for enhanced features
  # google_fonts: ^6.1.0  # For SF Pro Display font

flutter:
  uses-material-design: true
  
  # Optional: Add SF Pro Display font (or use Inter/Nunito Sans)
  # fonts:
  #   - family: SF Pro Display
  #     fonts:
  #       - asset: assets/fonts/SFProDisplay-Light.ttf
  #         weight: 300
  #       - asset: assets/fonts/SFProDisplay-Regular.ttf
  #         weight: 400
  #       - asset: assets/fonts/SFProDisplay-Semibold.ttf
  #         weight: 600
  #       - asset: assets/fonts/SFProDisplay-Bold.ttf
  #         weight: 700
```

### 3. Import in Your Files

```dart
import 'package:your_app/theme/antigravity_theme.dart';
```

---

## 🎨 Understanding the Design System

### Color System

```dart
// Usage Examples:
Container(
  color: AntiGravityColors.primary,        // Main brand color
  decoration: BoxDecoration(
    gradient: AntiGravityColors.primaryGradient,  // Gradient effect
  ),
)

Text(
  'Hello',
  style: TextStyle(color: AntiGravityColors.textPrimary),
)
```

**Available Colors:**
- `primary`, `primaryLight`, `primaryDark` - Brand colors
- `secondary`, `secondaryLight`, `secondaryDark` - Supporting colors
- `accent`, `accentPink` - Call-to-action colors
- `backgroundLight`, `backgroundDark` - Screen backgrounds
- `surfaceLight`, `surfaceDark` - Card/container backgrounds
- `textPrimary`, `textSecondary`, `textTertiary`, `textLight` - Text colors

### Typography System

```dart
Text(
  'Display Text',
  style: AntiGravityTextStyles.displayLarge,
)

Text(
  'Heading',
  style: AntiGravityTextStyles.headingMedium,
)

Text(
  'Body text',
  style: AntiGravityTextStyles.bodyMedium,
)
```

**Available Styles:**
- `displayLarge`, `displayMedium` - Hero text (34px, 28px)
- `headingLarge`, `headingMedium`, `headingSmall` - Headings (24px, 20px, 17px)
- `bodyLarge`, `bodyMedium`, `bodySmall` - Body text (17px, 15px, 13px)
- `caption` - Small text (11px)

### Spacing System (8pt Grid)

```dart
Padding(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),  // 16px
  child: Column(
    children: [
      SizedBox(height: AntiGravitySpacing.xl),      // 24px
      SizedBox(height: AntiGravitySpacing.md),      // 12px
    ],
  ),
)
```

**Available Spacing:**
- `xs` (4px), `sm` (8px), `md` (12px), `lg` (16px)
- `xl` (24px), `xxl` (32px), `xxxl` (48px), `huge` (64px)

### Shadow System (Anti-Gravity Effect)

```dart
Container(
  decoration: BoxDecoration(
    boxShadow: AntiGravityShadows.floatingShadow(),  // Default floating effect
    // OR
    boxShadow: AntiGravityShadows.elevatedShadow(),  // Higher elevation
    // OR
    boxShadow: AntiGravityShadows.glowShadow(
      color: AntiGravityColors.accent,               // Glowing effect
    ),
  ),
)
```

---

## 🔄 Step-by-Step Migration

### Phase 1: Update Theme (15-30 minutes)

**In your `main.dart`:**

```dart
import 'package:flutter/material.dart';
import 'theme/antigravity_theme.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Your App Name',
      theme: ThemeData(
        primaryColor: AntiGravityColors.primary,
        scaffoldBackgroundColor: AntiGravityColors.backgroundLight,
        colorScheme: const ColorScheme.light(
          primary: AntiGravityColors.primary,
          secondary: AntiGravityColors.secondary,
        ),
        textTheme: const TextTheme(
          displayLarge: AntiGravityTextStyles.displayLarge,
          headlineMedium: AntiGravityTextStyles.headingMedium,
          bodyLarge: AntiGravityTextStyles.bodyLarge,
          bodyMedium: AntiGravityTextStyles.bodyMedium,
        ),
        appBarTheme: AppBarTheme(
          elevation: 0,
          backgroundColor: Colors.transparent,
          foregroundColor: AntiGravityColors.textPrimary,
        ),
      ),
      home: const YourHomeScreen(),
    );
  }
}
```

### Phase 2: Replace Basic Widgets (1-2 hours)

#### Replace Cards

**BEFORE:**
```dart
Card(
  elevation: 4,
  child: Padding(
    padding: EdgeInsets.all(16),
    child: Text('Content'),
  ),
)
```

**AFTER:**
```dart
FloatingCard(
  child: Text('Content'),
)
```

#### Replace Buttons

**BEFORE:**
```dart
ElevatedButton(
  onPressed: () {},
  child: Text('Click Me'),
)
```

**AFTER:**
```dart
AntiGravityButton(
  text: 'Click Me',
  onPressed: () {},
  type: ButtonType.primary,
)
```

#### Replace Text Fields

**BEFORE:**
```dart
TextField(
  decoration: InputDecoration(
    labelText: 'Email',
    hintText: 'Enter email',
  ),
)
```

**AFTER:**
```dart
FloatingInputField(
  label: 'Email',
  hint: 'Enter email',
)
```

#### Replace Bottom Navigation

**BEFORE:**
```dart
BottomNavigationBar(
  currentIndex: _currentIndex,
  onTap: (index) => setState(() => _currentIndex = index),
  items: [
    BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
    BottomNavigationBarItem(icon: Icon(Icons.search), label: 'Search'),
  ],
)
```

**AFTER:**
```dart
FloatingTabBar(
  currentIndex: _currentIndex,
  onTap: (index) => setState(() => _currentIndex = index),
  items: const [
    FloatingTabItem(icon: Icons.home_outlined, activeIcon: Icons.home, label: 'Home'),
    FloatingTabItem(icon: Icons.search_outlined, activeIcon: Icons.search, label: 'Search'),
  ],
)
```

### Phase 3: Update Layouts (2-4 hours)

#### Add Proper Spacing

**BEFORE:**
```dart
Column(
  children: [
    Widget1(),
    SizedBox(height: 10),
    Widget2(),
  ],
)
```

**AFTER:**
```dart
Column(
  children: [
    Widget1(),
    SizedBox(height: AntiGravitySpacing.lg),  // Use 8pt grid
    Widget2(),
  ],
)
```

#### Update Container Styling

**BEFORE:**
```dart
Container(
  padding: EdgeInsets.all(20),
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(10),
    boxShadow: [
      BoxShadow(color: Colors.grey, blurRadius: 5),
    ],
  ),
)
```

**AFTER:**
```dart
Container(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  decoration: BoxDecoration(
    color: AntiGravityColors.surfaceLight,
    borderRadius: BorderRadius.circular(AntiGravityRadius.lg),
    boxShadow: AntiGravityShadows.floatingShadow(),
  ),
)
```

### Phase 4: Add Animations (2-3 hours)

Most components already have built-in animations, but you can add custom ones:

```dart
class MyFloatingWidget extends StatefulWidget {
  @override
  State<MyFloatingWidget> createState() => _MyFloatingWidgetState();
}

class _MyFloatingWidgetState extends State<MyFloatingWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: AntiGravityDurations.normal,
      vsync: this,
    )..repeat(reverse: true);

    _animation = Tween<double>(begin: 0, end: -8).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Transform.translate(
          offset: Offset(0, _animation.value),
          child: child,
        );
      },
      child: FloatingCard(child: Text('Floating!')),
    );
  }
}
```

---

## 🔀 Component Mapping

| Standard Flutter | Anti-Gravity Equivalent |
|-----------------|-------------------------|
| `Card` | `FloatingCard` |
| `ElevatedButton` | `AntiGravityButton(type: ButtonType.primary)` |
| `OutlinedButton` | `AntiGravityButton(type: ButtonType.secondary)` |
| `TextButton` | `AntiGravityButton(type: ButtonType.tertiary)` |
| `TextField` / `TextFormField` | `FloatingInputField` |
| `BottomNavigationBar` | `FloatingTabBar` |
| `Container` | Use `FloatingCard` for card-like containers |
| `ListTile` in `Card` | Wrap with `FloatingCard` |
| `Dialog` | Style with anti-gravity decorations |
| `BottomSheet` | Use rounded corners and floating shadows |

---

## 🎯 Common Patterns

### Pattern 1: List with Floating Items

```dart
ListView.separated(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  itemCount: items.length,
  separatorBuilder: (_, __) => SizedBox(height: AntiGravitySpacing.lg),
  itemBuilder: (context, index) {
    return FloatingCard(
      onTap: () => handleItemTap(index),
      child: ListTile(
        contentPadding: EdgeInsets.zero,
        leading: CircleAvatar(
          backgroundColor: AntiGravityColors.primary,
        ),
        title: Text(items[index].title),
        subtitle: Text(items[index].subtitle),
        trailing: Icon(Icons.arrow_forward_ios, size: 16),
      ),
    );
  },
)
```

### Pattern 2: Form Layout

```dart
Padding(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.stretch,
    children: [
      FloatingInputField(
        label: 'Email',
        hint: 'Enter your email',
        keyboardType: TextInputType.emailAddress,
        prefixIcon: Icon(Icons.email_outlined),
      ),
      SizedBox(height: AntiGravitySpacing.lg),
      FloatingInputField(
        label: 'Password',
        hint: 'Enter your password',
        obscureText: true,
        prefixIcon: Icon(Icons.lock_outlined),
      ),
      SizedBox(height: AntiGravitySpacing.xl),
      AntiGravityButton(
        text: 'Sign In',
        onPressed: handleSignIn,
        type: ButtonType.primary,
        size: ButtonSize.large,
        fullWidth: true,
      ),
    ],
  ),
)
```

### Pattern 3: Grid Layout

```dart
GridView.builder(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    crossAxisSpacing: AntiGravitySpacing.lg,
    mainAxisSpacing: AntiGravitySpacing.lg,
    childAspectRatio: 1.0,
  ),
  itemCount: items.length,
  itemBuilder: (context, index) {
    return FloatingCard(
      onTap: () => handleTap(index),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Container(
            width: 60,
            height: 60,
            decoration: BoxDecoration(
              gradient: AntiGravityColors.primaryGradient,
              borderRadius: BorderRadius.circular(AntiGravityRadius.lg),
            ),
            child: Icon(items[index].icon, color: Colors.white),
          ),
          SizedBox(height: AntiGravitySpacing.md),
          Text(
            items[index].title,
            style: AntiGravityTextStyles.bodyMedium,
            textAlign: TextAlign.center,
          ),
        ],
      ),
    );
  },
)
```

### Pattern 4: Settings Screen (iOS-style)

```dart
ListView(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  children: [
    Text(
      'Account',
      style: AntiGravityTextStyles.caption.copyWith(
        color: AntiGravityColors.textSecondary,
      ),
    ),
    SizedBox(height: AntiGravitySpacing.sm),
    FloatingCard(
      padding: EdgeInsets.zero,
      child: Column(
        children: [
          _buildSettingsTile('Profile', Icons.person_outlined),
          Divider(height: 1),
          _buildSettingsTile('Preferences', Icons.settings_outlined),
        ],
      ),
    ),
    SizedBox(height: AntiGravitySpacing.xl),
    Text(
      'Security',
      style: AntiGravityTextStyles.caption.copyWith(
        color: AntiGravityColors.textSecondary,
      ),
    ),
    SizedBox(height: AntiGravitySpacing.sm),
    FloatingCard(
      padding: EdgeInsets.zero,
      child: Column(
        children: [
          _buildSettingsTile('Privacy', Icons.lock_outlined),
          Divider(height: 1),
          _buildSettingsTile('Security', Icons.security_outlined),
        ],
      ),
    ),
  ],
)

Widget _buildSettingsTile(String title, IconData icon) {
  return ListTile(
    leading: Icon(icon, color: AntiGravityColors.primary),
    title: Text(title, style: AntiGravityTextStyles.bodyMedium),
    trailing: Icon(Icons.arrow_forward_ios, size: 16),
    onTap: () {},
  );
}
```

---

## ⚡ Performance Optimization

### 1. Use const Constructors

```dart
// ✅ GOOD
const FloatingCard(
  child: Text('Static content'),
)

// ❌ BAD
FloatingCard(
  child: Text('Static content'),
)
```

### 2. Limit Animation Controllers

Only use `SingleTickerProviderStateMixin` when you actually have animations:

```dart
// ✅ GOOD - Only when needed
class _MyWidgetState extends State<MyWidget> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  // ...
}

// ✅ GOOD - StatelessWidget for static content
class MyStaticCard extends StatelessWidget {
  // ...
}
```

### 3. Reuse Shadows and Decorations

```dart
// ✅ GOOD - Create once, reuse
final decoration = BoxDecoration(
  borderRadius: BorderRadius.circular(AntiGravityRadius.lg),
  boxShadow: AntiGravityShadows.floatingShadow(),
);

// Use in multiple places
Container(decoration: decoration)
Container(decoration: decoration)
```

### 4. Use ListView.builder for Long Lists

```dart
// ✅ GOOD - Efficient for long lists
ListView.builder(
  itemCount: 1000,
  itemBuilder: (context, index) => FloatingCard(...),
)

// ❌ BAD - Creates all items at once
ListView(
  children: List.generate(1000, (index) => FloatingCard(...)),
)
```

---

## 💡 Tips & Best Practices

### 1. Consistent Spacing
Always use the 8pt grid system:
```dart
// ✅ GOOD
EdgeInsets.all(AntiGravitySpacing.lg)        // 16px
SizedBox(height: AntiGravitySpacing.xl)      // 24px

// ❌ BAD
EdgeInsets.all(15)
SizedBox(height: 23)
```

### 2. Hierarchy with Typography
```dart
// Page title
Text('Dashboard', style: AntiGravityTextStyles.displayLarge)

// Section heading
Text('Recent Items', style: AntiGravityTextStyles.headingMedium)

// Body content
Text('Description', style: AntiGravityTextStyles.bodyMedium)

// Caption/helper text
Text('Last updated', style: AntiGravityTextStyles.caption)
```

### 3. Color Usage
```dart
// ✅ GOOD - Use semantic colors
Container(color: AntiGravityColors.surfaceLight)  // For cards
Text('Text', style: TextStyle(color: AntiGravityColors.textPrimary))

// ❌ BAD - Hardcoded colors
Container(color: Colors.white)
Text('Text', style: TextStyle(color: Colors.black))
```

### 4. Touch Targets
Ensure minimum 44px touch targets (iOS standard):
```dart
// ✅ GOOD
AntiGravityButton(
  text: 'Button',
  size: ButtonSize.medium,  // 44px height
  onPressed: () {},
)

// ✅ GOOD - Custom widget
GestureDetector(
  onTap: () {},
  child: Container(
    height: 44,  // Minimum touch target
    width: 44,
    child: Icon(Icons.favorite),
  ),
)
```

### 5. Accessibility

```dart
// Add semantic labels
Semantics(
  label: 'Like button',
  button: true,
  child: AntiGravityButton(...),
)

// Ensure sufficient contrast (WCAG AA: 4.5:1 for text)
// The theme already follows this, but verify custom colors:
Text(
  'Text',
  style: TextStyle(
    color: AntiGravityColors.textPrimary,  // ✅ High contrast
  ),
)
```

### 6. Dark Mode Support

```dart
// Use theme-aware colors
final backgroundColor = Theme.of(context).brightness == Brightness.dark
    ? AntiGravityColors.backgroundDark
    : AntiGravityColors.backgroundLight;

// Or let the theme handle it automatically
Container(
  color: Theme.of(context).scaffoldBackgroundColor,
)
```

### 7. Test on Different Devices

```dart
// Use MediaQuery for responsive design
final screenWidth = MediaQuery.of(context).size.width;

// Adjust spacing for smaller screens
final padding = screenWidth < 360 
    ? AntiGravitySpacing.md 
    : AntiGravitySpacing.lg;
```

---

## 🎓 Learning Resources

### Example Apps to Reference
1. **iOS Settings App** - Grouped list style with floating cards
2. **iOS Messages** - Floating bubbles with elevation
3. **iOS Weather** - Gradient backgrounds with floating cards
4. **iOS Photos** - Grid layout with hover effects

### Flutter Documentation
- [Material Design Components](https://docs.flutter.dev/development/ui/widgets/material)
- [Animations](https://docs.flutter.dev/development/ui/animations)
- [Themes](https://docs.flutter.dev/cookbook/design/themes)

---

## 📞 Need Help?

If you encounter issues:
1. Check that you've imported the theme file correctly
2. Verify widget hierarchy (FloatingCard needs proper parent)
3. Look at the usage_example.dart for complete working examples
4. Test on a physical device (animations look better than emulator)

---

## ✅ Migration Checklist

- [ ] Added antigravity_theme.dart to project
- [ ] Updated MaterialApp theme in main.dart
- [ ] Replaced Card widgets with FloatingCard
- [ ] Replaced buttons with AntiGravityButton
- [ ] Replaced TextFields with FloatingInputField
- [ ] Updated spacing to use 8pt grid system
- [ ] Updated colors to use AntiGravityColors
- [ ] Updated text styles to use AntiGravityTextStyles
- [ ] Replaced BottomNavigationBar with FloatingTabBar
- [ ] Added proper shadows/elevation to containers
- [ ] Tested on both light and dark modes
- [ ] Tested on different screen sizes
- [ ] Verified accessibility (contrast, touch targets)
- [ ] Optimized animations (const, controllers)

---

**Estimated Total Migration Time:** 6-12 hours for a medium-sized app (20-30 screens)

Good luck with your migration! The anti-gravity design will make your app feel modern, polished, and iOS-native.
