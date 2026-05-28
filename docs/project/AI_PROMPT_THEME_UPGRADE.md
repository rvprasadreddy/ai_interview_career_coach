# AI PROMPT: Anti-Gravity Theme Upgradation for Existing Flutter App

## Context
You are an expert Flutter developer tasked with upgrading an existing Flutter mobile application to use the Anti-Gravity design system. The Anti-Gravity design system is inspired by iOS design language with a weightless, floating aesthetic featuring ethereal colors, smooth animations, and elevated UI components.

## Your Mission
Analyze the provided Flutter app codebase and systematically upgrade ALL screens to the Anti-Gravity theme while maintaining existing functionality and improving user experience.

---

## Design System Reference

### Available Components

#### 1. FloatingCard
Replace all `Card`, `Container` (when used as cards), and similar widgets.
```dart
FloatingCard(
  onTap: () {},  // Optional
  padding: EdgeInsets.all(16),  // Optional
  child: YourContent(),
)
```

#### 2. AntiGravityButton
Replace all buttons (ElevatedButton, OutlinedButton, TextButton).
```dart
AntiGravityButton(
  text: 'Button Text',
  onPressed: () {},
  type: ButtonType.primary,  // primary, secondary, tertiary, accent
  size: ButtonSize.medium,   // small, medium, large
  icon: Icons.star,          // Optional
  fullWidth: true,           // Optional
)
```

#### 3. FloatingInputField
Replace all TextField and TextFormField.
```dart
FloatingInputField(
  label: 'Email',
  hint: 'Enter your email',
  controller: _controller,
  keyboardType: TextInputType.emailAddress,
  obscureText: false,
  prefixIcon: Icon(Icons.email_outlined),  // Optional
  suffixIcon: Icon(Icons.check),           // Optional
  validator: (value) => validation,         // Optional
)
```

#### 4. FloatingTabBar
Replace BottomNavigationBar.
```dart
FloatingTabBar(
  currentIndex: _currentIndex,
  onTap: (index) => setState(() => _currentIndex = index),
  items: const [
    FloatingTabItem(
      icon: Icons.home_outlined,
      activeIcon: Icons.home,
      label: 'Home',
    ),
    // Add more items...
  ],
)
```

### Color Palette
```dart
// Primary Colors
AntiGravityColors.primary         // Main brand color (#5E7CE2)
AntiGravityColors.primaryLight    // Lighter variant
AntiGravityColors.primaryDark     // Darker variant

// Secondary Colors
AntiGravityColors.secondary       // Supporting color (#A78BFA)
AntiGravityColors.secondaryLight
AntiGravityColors.secondaryDark

// Accent Colors
AntiGravityColors.accent          // Holographic cyan (#00F5FF)
AntiGravityColors.accentPink      // Holographic pink (#FF3B81)

// Backgrounds
AntiGravityColors.backgroundLight // Screen background (#F8F9FF)
AntiGravityColors.backgroundDark  // Dark mode background (#0F1115)
AntiGravityColors.surfaceLight    // Card/surface background (#FFFFFF)
AntiGravityColors.surfaceDark     // Dark mode surface (#1A1D24)

// Text Colors
AntiGravityColors.textPrimary     // Main text (#1A1D24)
AntiGravityColors.textSecondary   // Secondary text (#6B7280)
AntiGravityColors.textTertiary    // Tertiary/disabled text (#9CA3AF)
AntiGravityColors.textLight       // Light text for dark backgrounds (#FFFFFF)

// Gradients
AntiGravityColors.primaryGradient // Blue to purple gradient
AntiGravityColors.accentGradient  // Cyan to purple gradient
```

### Typography
```dart
// Display (Hero text)
AntiGravityTextStyles.displayLarge   // 34px, Bold
AntiGravityTextStyles.displayMedium  // 28px, Semibold

// Headings
AntiGravityTextStyles.headingLarge   // 24px, Semibold
AntiGravityTextStyles.headingMedium  // 20px, Semibold
AntiGravityTextStyles.headingSmall   // 17px, Semibold

// Body Text
AntiGravityTextStyles.bodyLarge      // 17px, Regular
AntiGravityTextStyles.bodyMedium     // 15px, Regular
AntiGravityTextStyles.bodySmall      // 13px, Regular

// Caption
AntiGravityTextStyles.caption        // 11px, Regular
```

### Spacing (8pt Grid)
```dart
AntiGravitySpacing.xs      // 4px
AntiGravitySpacing.sm      // 8px
AntiGravitySpacing.md      // 12px
AntiGravitySpacing.lg      // 16px  (standard screen margin)
AntiGravitySpacing.xl      // 24px
AntiGravitySpacing.xxl     // 32px
AntiGravitySpacing.xxxl    // 48px
AntiGravitySpacing.huge    // 64px
```

### Shadows & Elevation
```dart
// Floating effect (use for cards, buttons)
boxShadow: AntiGravityShadows.floatingShadow()

// Higher elevation (use for modals, dialogs)
boxShadow: AntiGravityShadows.elevatedShadow()

// Glow effect (use for accent elements)
boxShadow: AntiGravityShadows.glowShadow(color: AntiGravityColors.accent)
```

### Border Radius
```dart
AntiGravityRadius.sm      // 8px
AntiGravityRadius.md      // 12px  (standard for buttons, cards)
AntiGravityRadius.lg      // 16px
AntiGravityRadius.xl      // 20px
AntiGravityRadius.xxl     // 24px
AntiGravityRadius.full    // 9999px (for pills, circular elements)
```

### Animation Durations
```dart
AntiGravityDurations.fast      // 200ms (micro-interactions)
AntiGravityDurations.normal    // 300ms (standard animations)
AntiGravityDurations.slow      // 400ms (emphasis animations)
AntiGravityDurations.verySlow  // 600ms (page transitions)
```

---

## Upgrade Instructions

### Step 1: Analyze Current Screen Structure
For each screen file provided:
1. Identify the screen type (List, Form, Detail, Dashboard, Settings, etc.)
2. List all interactive elements (buttons, cards, inputs, navigation)
3. Note the current color scheme and spacing patterns
4. Identify any custom animations or transitions

### Step 2: Apply Theme Systematically

#### A. Update Imports
Add at the top of each file:
```dart
import 'package:your_app/theme/antigravity_theme.dart';
```

#### B. Update Scaffold
```dart
// BEFORE
Scaffold(
  backgroundColor: Colors.white,
  // ...
)

// AFTER
Scaffold(
  backgroundColor: AntiGravityColors.backgroundLight,
  extendBody: true,  // For floating tab bar
  // ...
)
```

#### C. Update AppBar
```dart
// BEFORE
AppBar(
  title: Text('Screen Title'),
  backgroundColor: Colors.blue,
  elevation: 4,
)

// AFTER
AppBar(
  title: Text(
    'Screen Title',
    style: AntiGravityTextStyles.headingMedium,
  ),
  backgroundColor: Colors.transparent,
  elevation: 0,
  foregroundColor: AntiGravityColors.textPrimary,
  centerTitle: true,  // iOS style
)
```

#### D. Replace All Cards
```dart
// BEFORE
Card(
  elevation: 2,
  child: Padding(
    padding: EdgeInsets.all(16),
    child: Column(
      children: [
        Text('Title'),
        Text('Subtitle'),
      ],
    ),
  ),
)

// AFTER
FloatingCard(
  onTap: () {},  // Add if card should be tappable
  child: Column(
    children: [
      Text(
        'Title',
        style: AntiGravityTextStyles.headingSmall,
      ),
      SizedBox(height: AntiGravitySpacing.sm),
      Text(
        'Subtitle',
        style: AntiGravityTextStyles.bodyMedium.copyWith(
          color: AntiGravityColors.textSecondary,
        ),
      ),
    ],
  ),
)
```

#### E. Replace All Buttons
```dart
// BEFORE - Primary Button
ElevatedButton(
  onPressed: () {},
  child: Text('Submit'),
)

// AFTER
AntiGravityButton(
  text: 'Submit',
  onPressed: () {},
  type: ButtonType.primary,
  size: ButtonSize.large,
  fullWidth: true,  // For full-width buttons
)

// BEFORE - Outlined Button
OutlinedButton(
  onPressed: () {},
  child: Text('Cancel'),
)

// AFTER
AntiGravityButton(
  text: 'Cancel',
  onPressed: () {},
  type: ButtonType.secondary,
)

// BEFORE - Text Button
TextButton(
  onPressed: () {},
  child: Text('Skip'),
)

// AFTER
AntiGravityButton(
  text: 'Skip',
  onPressed: () {},
  type: ButtonType.tertiary,
)
```

#### F. Replace All Text Fields
```dart
// BEFORE
TextField(
  decoration: InputDecoration(
    labelText: 'Email',
    hintText: 'Enter email',
    prefixIcon: Icon(Icons.email),
  ),
  controller: _emailController,
)

// AFTER
FloatingInputField(
  label: 'Email',
  hint: 'Enter email',
  controller: _emailController,
  keyboardType: TextInputType.emailAddress,
  prefixIcon: Icon(
    Icons.email_outlined,
    color: AntiGravityColors.textSecondary,
  ),
)
```

#### G. Update List Views
```dart
// BEFORE
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return Card(
      child: ListTile(
        title: Text(items[index].title),
      ),
    );
  },
)

// AFTER
ListView.separated(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  itemCount: items.length,
  separatorBuilder: (_, __) => SizedBox(height: AntiGravitySpacing.lg),
  itemBuilder: (context, index) {
    return FloatingCard(
      onTap: () => handleItemTap(index),
      child: ListTile(
        contentPadding: EdgeInsets.zero,
        leading: Container(
          width: 48,
          height: 48,
          decoration: BoxDecoration(
            gradient: AntiGravityColors.primaryGradient,
            borderRadius: BorderRadius.circular(AntiGravityRadius.md),
          ),
          child: Icon(
            items[index].icon,
            color: Colors.white,
          ),
        ),
        title: Text(
          items[index].title,
          style: AntiGravityTextStyles.bodyLarge.copyWith(
            fontWeight: FontWeight.w600,
          ),
        ),
        subtitle: Text(
          items[index].subtitle,
          style: AntiGravityTextStyles.bodySmall.copyWith(
            color: AntiGravityColors.textSecondary,
          ),
        ),
        trailing: Icon(
          Icons.arrow_forward_ios,
          size: 16,
          color: AntiGravityColors.textTertiary,
        ),
      ),
    );
  },
)
```

#### H. Update Grid Views
```dart
// BEFORE
GridView.count(
  crossAxisCount: 2,
  children: items.map((item) => Card(...)).toList(),
)

// AFTER
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
      child: // Your content
    );
  },
)
```

#### I. Update Bottom Navigation
```dart
// BEFORE
bottomNavigationBar: BottomNavigationBar(
  currentIndex: _currentIndex,
  onTap: (index) => setState(() => _currentIndex = index),
  items: [
    BottomNavigationBarItem(
      icon: Icon(Icons.home),
      label: 'Home',
    ),
    // ...
  ],
)

// AFTER
bottomNavigationBar: FloatingTabBar(
  currentIndex: _currentIndex,
  onTap: (index) => setState(() => _currentIndex = index),
  items: const [
    FloatingTabItem(
      icon: Icons.home_outlined,
      activeIcon: Icons.home,
      label: 'Home',
    ),
    FloatingTabItem(
      icon: Icons.search_outlined,
      activeIcon: Icons.search,
      label: 'Search',
    ),
    FloatingTabItem(
      icon: Icons.favorite_outline,
      activeIcon: Icons.favorite,
      label: 'Favorites',
    ),
    FloatingTabItem(
      icon: Icons.person_outline,
      activeIcon: Icons.person,
      label: 'Profile',
    ),
  ],
)
```

#### J. Update Spacing
Replace all hardcoded spacing with the 8pt grid:
```dart
// BEFORE
Padding(padding: EdgeInsets.all(15))
SizedBox(height: 10)
SizedBox(width: 20)

// AFTER
Padding(padding: EdgeInsets.all(AntiGravitySpacing.lg))  // 16px
SizedBox(height: AntiGravitySpacing.sm)                   // 8px
SizedBox(width: AntiGravitySpacing.xl)                    // 24px
```

#### K. Update Container Decorations
```dart
// BEFORE
Container(
  padding: EdgeInsets.all(20),
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(10),
    boxShadow: [
      BoxShadow(
        color: Colors.grey.withOpacity(0.3),
        blurRadius: 5,
        offset: Offset(0, 2),
      ),
    ],
  ),
)

// AFTER
Container(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  decoration: BoxDecoration(
    color: AntiGravityColors.surfaceLight,
    borderRadius: BorderRadius.circular(AntiGravityRadius.lg),
    boxShadow: AntiGravityShadows.floatingShadow(),
  ),
)
```

#### L. Update Text Styles
```dart
// BEFORE
Text(
  'Heading',
  style: TextStyle(
    fontSize: 24,
    fontWeight: FontWeight.bold,
  ),
)

// AFTER
Text(
  'Heading',
  style: AntiGravityTextStyles.headingLarge.copyWith(
    color: AntiGravityColors.textPrimary,
  ),
)
```

#### M. Update Dialogs and Bottom Sheets
```dart
// For Dialogs
showDialog(
  context: context,
  builder: (context) => Dialog(
    backgroundColor: Colors.transparent,
    child: Container(
      padding: EdgeInsets.all(AntiGravitySpacing.xl),
      decoration: BoxDecoration(
        color: AntiGravityColors.surfaceLight,
        borderRadius: BorderRadius.circular(AntiGravityRadius.xxl),
        boxShadow: AntiGravityShadows.elevatedShadow(),
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(
            'Dialog Title',
            style: AntiGravityTextStyles.headingLarge,
          ),
          SizedBox(height: AntiGravitySpacing.lg),
          Text(
            'Dialog message',
            style: AntiGravityTextStyles.bodyMedium.copyWith(
              color: AntiGravityColors.textSecondary,
            ),
          ),
          SizedBox(height: AntiGravitySpacing.xl),
          Row(
            children: [
              Expanded(
                child: AntiGravityButton(
                  text: 'Cancel',
                  onPressed: () => Navigator.pop(context),
                  type: ButtonType.secondary,
                ),
              ),
              SizedBox(width: AntiGravitySpacing.md),
              Expanded(
                child: AntiGravityButton(
                  text: 'Confirm',
                  onPressed: () => Navigator.pop(context),
                  type: ButtonType.primary,
                ),
              ),
            ],
          ),
        ],
      ),
    ),
  ),
);

// For Bottom Sheets
showModalBottomSheet(
  context: context,
  backgroundColor: Colors.transparent,
  isScrollControlled: true,
  builder: (context) => Container(
    margin: EdgeInsets.all(AntiGravitySpacing.lg),
    padding: EdgeInsets.all(AntiGravitySpacing.xl),
    decoration: BoxDecoration(
      color: AntiGravityColors.surfaceLight,
      borderRadius: BorderRadius.circular(AntiGravityRadius.xxl),
      boxShadow: AntiGravityShadows.elevatedShadow(),
    ),
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        // Handle bar
        Container(
          width: 40,
          height: 4,
          decoration: BoxDecoration(
            color: AntiGravityColors.textTertiary.withOpacity(0.3),
            borderRadius: BorderRadius.circular(AntiGravityRadius.full),
          ),
        ),
        SizedBox(height: AntiGravitySpacing.xl),
        // Your content
      ],
    ),
  ),
);
```

### Step 3: Screen-Specific Patterns

#### Login/Auth Screens
```dart
// Use FloatingCard for form container
FloatingCard(
  padding: EdgeInsets.all(AntiGravitySpacing.xl),
  child: Column(
    children: [
      // Logo with gradient background
      Container(
        width: 80,
        height: 80,
        decoration: BoxDecoration(
          gradient: AntiGravityColors.primaryGradient,
          shape: BoxShape.circle,
          boxShadow: AntiGravityShadows.glowShadow(
            color: AntiGravityColors.primary,
          ),
        ),
        child: Icon(Icons.lock_outline, color: Colors.white, size: 40),
      ),
      SizedBox(height: AntiGravitySpacing.xl),
      
      FloatingInputField(
        label: 'Email',
        hint: 'Enter your email',
        controller: _emailController,
        keyboardType: TextInputType.emailAddress,
        prefixIcon: Icon(Icons.email_outlined),
      ),
      SizedBox(height: AntiGravitySpacing.lg),
      
      FloatingInputField(
        label: 'Password',
        hint: 'Enter your password',
        controller: _passwordController,
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

#### Dashboard/Home Screens
```dart
// Use SingleChildScrollView with multiple FloatingCard sections
SingleChildScrollView(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      // Welcome section
      Text(
        'Welcome back!',
        style: AntiGravityTextStyles.displayMedium,
      ),
      SizedBox(height: AntiGravitySpacing.xl),
      
      // Stats cards in grid
      GridView.count(
        shrinkWrap: true,
        physics: NeverScrollableScrollPhysics(),
        crossAxisCount: 2,
        crossAxisSpacing: AntiGravitySpacing.lg,
        mainAxisSpacing: AntiGravitySpacing.lg,
        childAspectRatio: 1.2,
        children: [
          FloatingCard(
            child: _buildStatCard('Total', '1,234', Icons.trending_up),
          ),
          FloatingCard(
            child: _buildStatCard('Active', '567', Icons.check_circle),
          ),
        ],
      ),
      SizedBox(height: AntiGravitySpacing.xl),
      
      // Recent items section
      Text(
        'Recent Activity',
        style: AntiGravityTextStyles.headingMedium,
      ),
      SizedBox(height: AntiGravitySpacing.lg),
      
      // List of recent items
      ...recentItems.map((item) => Padding(
        padding: EdgeInsets.only(bottom: AntiGravitySpacing.lg),
        child: FloatingCard(
          onTap: () => handleItemTap(item),
          child: // Item content
        ),
      )),
    ],
  ),
)
```

#### Settings Screens (iOS-style)
```dart
ListView(
  padding: EdgeInsets.all(AntiGravitySpacing.lg),
  children: [
    // Section header
    Padding(
      padding: EdgeInsets.only(
        left: AntiGravitySpacing.sm,
        bottom: AntiGravitySpacing.sm,
      ),
      child: Text(
        'ACCOUNT',
        style: AntiGravityTextStyles.caption.copyWith(
          color: AntiGravityColors.textSecondary,
          fontWeight: FontWeight.w600,
        ),
      ),
    ),
    
    // Grouped settings
    FloatingCard(
      padding: EdgeInsets.zero,
      child: Column(
        children: [
          _buildSettingsTile('Profile', Icons.person_outline, () {}),
          Divider(height: 1),
          _buildSettingsTile('Preferences', Icons.settings_outlined, () {}),
          Divider(height: 1),
          _buildSettingsTile('Privacy', Icons.lock_outlined, () {}),
        ],
      ),
    ),
    SizedBox(height: AntiGravitySpacing.xl),
    
    // Another section...
  ],
)

Widget _buildSettingsTile(String title, IconData icon, VoidCallback onTap) {
  return ListTile(
    contentPadding: EdgeInsets.symmetric(
      horizontal: AntiGravitySpacing.lg,
      vertical: AntiGravitySpacing.sm,
    ),
    leading: Container(
      width: 32,
      height: 32,
      decoration: BoxDecoration(
        color: AntiGravityColors.primary.withOpacity(0.1),
        borderRadius: BorderRadius.circular(AntiGravityRadius.sm),
      ),
      child: Icon(icon, color: AntiGravityColors.primary, size: 20),
    ),
    title: Text(title, style: AntiGravityTextStyles.bodyMedium),
    trailing: Icon(
      Icons.arrow_forward_ios,
      size: 16,
      color: AntiGravityColors.textTertiary,
    ),
    onTap: onTap,
  );
}
```

#### Profile Screens
```dart
Column(
  children: [
    // Header with gradient background
    Container(
      height: 200,
      decoration: BoxDecoration(
        gradient: AntiGravityColors.primaryGradient,
      ),
      child: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Container(
              width: 100,
              height: 100,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                border: Border.all(color: Colors.white, width: 4),
                boxShadow: AntiGravityShadows.elevatedShadow(),
              ),
              child: CircleAvatar(
                backgroundColor: Colors.white,
                child: Text(
                  'JD',
                  style: AntiGravityTextStyles.displayMedium.copyWith(
                    color: AntiGravityColors.primary,
                  ),
                ),
              ),
            ),
            SizedBox(height: AntiGravitySpacing.md),
            Text(
              'John Doe',
              style: AntiGravityTextStyles.headingLarge.copyWith(
                color: Colors.white,
              ),
            ),
          ],
        ),
      ),
    ),
    
    // Profile details
    Expanded(
      child: ListView(
        padding: EdgeInsets.all(AntiGravitySpacing.lg),
        children: [
          FloatingCard(
            child: // Profile info
          ),
          SizedBox(height: AntiGravitySpacing.lg),
          // More cards...
        ],
      ),
    ),
  ],
)
```

#### Detail Screens
```dart
// Use CustomScrollView with SliverAppBar for hero effect
CustomScrollView(
  slivers: [
    SliverAppBar(
      expandedHeight: 200,
      pinned: true,
      flexibleSpace: FlexibleSpaceBar(
        background: Container(
          decoration: BoxDecoration(
            gradient: AntiGravityColors.primaryGradient,
          ),
        ),
        title: Text(
          'Item Title',
          style: AntiGravityTextStyles.headingMedium.copyWith(
            color: Colors.white,
          ),
        ),
      ),
    ),
    SliverToBoxAdapter(
      child: Padding(
        padding: EdgeInsets.all(AntiGravitySpacing.lg),
        child: Column(
          children: [
            FloatingCard(
              child: // Details content
            ),
            SizedBox(height: AntiGravitySpacing.lg),
            // More cards...
          ],
        ),
      ),
    ),
  ],
)
```

---

## Output Format

For each screen file, provide:

### 1. Screen Analysis
```
Screen Name: [e.g., LoginScreen]
Current Type: [e.g., Form-based authentication screen]
Key Components: [List main interactive elements]
Upgrade Priority: [High/Medium/Low based on user visibility]
```

### 2. Updated Code
Provide the COMPLETE updated Dart file with:
- All imports updated
- All widgets replaced with Anti-Gravity components
- All colors updated to use AntiGravityColors
- All spacing updated to use 8pt grid
- All typography updated to use AntiGravityTextStyles
- Comments explaining major changes

### 3. Before/After Comparison
Highlight the key visual differences:
```
BEFORE:
- Standard Material Design Card with grey shadows
- Blue ElevatedButton
- Basic TextField with underline

AFTER:
- FloatingCard with ethereal blue shadow and elevation
- AntiGravityButton with gradient and glow effect
- FloatingInputField with floating label animation
```

### 4. Testing Checklist
For each screen:
- [ ] All buttons are tappable and have proper animations
- [ ] Cards have floating effect on tap/hover
- [ ] Text is readable with proper contrast (WCAG AA)
- [ ] Spacing follows 8pt grid system
- [ ] Navigation works correctly
- [ ] Forms validate properly
- [ ] Loading states use Anti-Gravity styling
- [ ] Error states use Anti-Gravity styling

---

## Special Considerations

### 1. Maintain Functionality
- Do NOT change any business logic
- Keep all navigation flows intact
- Preserve all state management
- Keep all API calls and data handling unchanged
- Only update UI/UX presentation layer

### 2. Accessibility
- Ensure all touch targets are minimum 44px (iOS standard)
- Maintain WCAG AA contrast ratios (4.5:1 for text)
- Add Semantics widgets where appropriate
- Support dark mode (use theme-aware colors)

### 3. Performance
- Use `const` constructors where possible
- Avoid creating new instances of shadows/gradients repeatedly
- Use ListView.builder for long lists
- Implement proper disposal of animation controllers

### 4. Consistency
- Use the same button type for similar actions across screens
- Use consistent spacing throughout the app
- Apply the same card style for similar content types
- Use consistent icon styles (outlined vs filled)

### 5. iOS Design Principles
- Center titles in AppBars
- Use floating tab bar at bottom (not flush with screen edge)
- Group related settings in cards
- Use light touch feedback animations
- Implement pull-to-refresh where appropriate
- Use proper iOS-style modals and sheets

---

## Example Upgrade Request Format

When requesting an upgrade, provide:

```
SCREEN FILE: lib/screens/home_screen.dart

CURRENT CODE:
[Paste your current screen code]

DESIRED OUTCOME:
- Convert to Anti-Gravity theme
- Maintain all existing functionality
- Add smooth animations to card taps
- Improve visual hierarchy with proper typography

SPECIFIC REQUIREMENTS:
- Keep the search functionality working
- Preserve the infinite scroll
- Update the filter buttons to use AntiGravityButton
```

---

## Success Criteria

A successful upgrade should:
1. ✅ Maintain 100% of original functionality
2. ✅ Use Anti-Gravity components throughout
3. ✅ Follow 8pt grid spacing system
4. ✅ Use semantic color palette consistently
5. ✅ Have smooth, physics-based animations
6. ✅ Meet WCAG AA accessibility standards
7. ✅ Look iOS-native and polished
8. ✅ Have zero compilation errors
9. ✅ Pass all existing tests
10. ✅ Improve perceived performance with animations

---

## Common Pitfalls to Avoid

1. ❌ Don't mix old and new styles in the same screen
2. ❌ Don't use hardcoded colors or spacing values
3. ❌ Don't break existing navigation or state management
4. ❌ Don't remove important user feedback (loading states, errors)
5. ❌ Don't make buttons too small (< 44px touch target)
6. ❌ Don't forget to update dialogs and bottom sheets
7. ❌ Don't ignore dark mode support
8. ❌ Don't create memory leaks with animation controllers
9. ❌ Don't sacrifice performance for aesthetics
10. ❌ Don't forget to test on different screen sizes

---

## Ready to Upgrade!

Provide your screen files one at a time or in batches, and I will:
1. Analyze the current implementation
2. Identify all components that need upgrading
3. Provide the complete updated code
4. Explain all changes made
5. Highlight potential issues or improvements
6. Give you a testing checklist

Let's transform your Flutter app with the Anti-Gravity theme! 🚀
