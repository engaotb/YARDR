# YARDR - Complete UI/UX Design System

## Design Principles

### Platform-Specific Guidelines

#### iOS (Liquid Glass UI)
- Translucent backgrounds with blur effects
- Smooth, spring-based animations
- SF Symbols for icons
- Large, readable typography
- Depth through layering
- Natural gestures

#### Android (Material Design 3)
- Dynamic color theming
- Material You personalization
- Elevated surfaces
- Ripple effects
- FAB for primary actions
- Bottom app bars

#### Web (ShadCN)
- Clean, modern interface
- Responsive grid system
- Consistent spacing
- Accessible components
- Keyboard navigation
- Dark mode support

## Color System

```typescript
// colors.ts
export const colors = {
  // Primary Colors
  primary: {
    50: '#E6F3FF',
    100: '#CCE7FF',
    200: '#99CFFF',
    300: '#66B7FF',
    400: '#339FFF',
    500: '#0087FF', // Main
    600: '#006FCC',
    700: '#005799',
    800: '#003F66',
    900: '#002733'
  },
  
  // Secondary Colors
  secondary: {
    50: '#F0FDF4',
    100: '#DCFCE7',
    200: '#BBF7D0',
    300: '#86EFAC',
    400: '#4ADE80',
    500: '#22C55E', // Main
    600: '#16A34A',
    700: '#15803D',
    800: '#166534',
    900: '#14532D'
  },
  
  // Semantic Colors
  success: '#10B981',
  warning: '#F59E0B',
  error: '#EF4444',
  info: '#3B82F6',
  
  // Neutral Colors
  neutral: {
    50: '#FAFAFA',
    100: '#F5F5F5',
    200: '#E5E5E5',
    300: '#D4D4D4',
    400: '#A3A3A3',
    500: '#737373',
    600: '#525252',
    700: '#404040',
    800: '#262626',
    900: '#171717'
  },
  
  // Role-Specific Colors
  roles: {
    renter: '#0087FF',
    owner: '#22C55E',
    driver: '#F59E0B',
    admin: '#8B5CF6'
  }
};
```

## Typography System

### iOS Typography (SF Pro)

```typescript
// typography.ts
export const typography = {
  ios: {
    largeTitle: {
      fontFamily: 'SF Pro Display',
      fontSize: 34,
      fontWeight: '700',
      lineHeight: 41
    },
    title1: {
      fontFamily: 'SF Pro Display',
      fontSize: 28,
      fontWeight: '700',
      lineHeight: 34
    },
    title2: {
      fontFamily: 'SF Pro Display',
      fontSize: 22,
      fontWeight: '700',
      lineHeight: 28
    },
    title3: {
      fontFamily: 'SF Pro Display',
      fontSize: 20,
      fontWeight: '600',
      lineHeight: 25
    },
    headline: {
      fontFamily: 'SF Pro Text',
      fontSize: 17,
      fontWeight: '600',
      lineHeight: 22
    },
    body: {
      fontFamily: 'SF Pro Text',
      fontSize: 17,
      fontWeight: '400',
      lineHeight: 22
    },
    callout: {
      fontFamily: 'SF Pro Text',
      fontSize: 16,
      fontWeight: '400',
      lineHeight: 21
    },
    subheadline: {
      fontFamily: 'SF Pro Text',
      fontSize: 15,
      fontWeight: '400',
      lineHeight: 20
    },
    footnote: {
      fontFamily: 'SF Pro Text',
      fontSize: 13,
      fontWeight: '400',
      lineHeight: 18
    },
    caption1: {
      fontFamily: 'SF Pro Text',
      fontSize: 12,
      fontWeight: '400',
      lineHeight: 16
    },
    caption2: {
      fontFamily: 'SF Pro Text',
      fontSize: 11,
      fontWeight: '400',
      lineHeight: 13
    }
  }
};
```

### Android Typography (Roboto)

```typescript
export const typography = {
  android: {
    displayLarge: {
      fontFamily: 'Roboto',
      fontSize: 57,
      fontWeight: '400',
      lineHeight: 64
    },
    displayMedium: {
      fontFamily: 'Roboto',
      fontSize: 45,
      fontWeight: '400',
      lineHeight: 52
    },
    displaySmall: {
      fontFamily: 'Roboto',
      fontSize: 36,
      fontWeight: '400',
      lineHeight: 44
    },
    headlineLarge: {
      fontFamily: 'Roboto',
      fontSize: 32,
      fontWeight: '400',
      lineHeight: 40
    },
    headlineMedium: {
      fontFamily: 'Roboto',
      fontSize: 28,
      fontWeight: '400',
      lineHeight: 36
    },
    headlineSmall: {
      fontFamily: 'Roboto',
      fontSize: 24,
      fontWeight: '400',
      lineHeight: 32
    },
    titleLarge: {
      fontFamily: 'Roboto',
      fontSize: 22,
      fontWeight: '500',
      lineHeight: 28
    },
    titleMedium: {
      fontFamily: 'Roboto',
      fontSize: 16,
      fontWeight: '500',
      lineHeight: 24
    },
    titleSmall: {
      fontFamily: 'Roboto',
      fontSize: 14,
      fontWeight: '500',
      lineHeight: 20
    },
    bodyLarge: {
      fontFamily: 'Roboto',
      fontSize: 16,
      fontWeight: '400',
      lineHeight: 24
    },
    bodyMedium: {
      fontFamily: 'Roboto',
      fontSize: 14,
      fontWeight: '400',
      lineHeight: 20
    },
    bodySmall: {
      fontFamily: 'Roboto',
      fontSize: 12,
      fontWeight: '400',
      lineHeight: 16
    },
    labelLarge: {
      fontFamily: 'Roboto',
      fontSize: 14,
      fontWeight: '500',
      lineHeight: 20
    },
    labelMedium: {
      fontFamily: 'Roboto',
      fontSize: 12,
      fontWeight: '500',
      lineHeight: 16
    },
    labelSmall: {
      fontFamily: 'Roboto',
      fontSize: 11,
      fontWeight: '500',
      lineHeight: 16
    }
  }
};
```

## Component Library

### Button Component

```typescript
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
  size: 'small' | 'medium' | 'large';
  fullWidth?: boolean;
  loading?: boolean;
  disabled?: boolean;
  icon?: React.ReactNode;
  children: React.ReactNode;
  onPress: () => void;
}

// Button Variants
const buttonVariants = {
  primary: {
    backgroundColor: colors.primary[500],
    color: 'white',
    borderWidth: 0
  },
  secondary: {
    backgroundColor: colors.secondary[500],
    color: 'white',
    borderWidth: 0
  },
  outline: {
    backgroundColor: 'transparent',
    color: colors.primary[500],
    borderWidth: 1,
    borderColor: colors.primary[500]
  },
  ghost: {
    backgroundColor: 'transparent',
    color: colors.primary[500],
    borderWidth: 0
  },
  danger: {
    backgroundColor: colors.error,
    color: 'white',
    borderWidth: 0
  }
};

// Button Sizes
const buttonSizes = {
  small: {
    paddingHorizontal: 12,
    paddingVertical: 8,
    fontSize: 14
  },
  medium: {
    paddingHorizontal: 16,
    paddingVertical: 12,
    fontSize: 16
  },
  large: {
    paddingHorizontal: 24,
    paddingVertical: 16,
    fontSize: 18
  }
};
```

### Card Component

```typescript
interface CardProps {
  variant: 'elevated' | 'outlined' | 'filled';
  padding: 'none' | 'small' | 'medium' | 'large';
  children: React.ReactNode;
}

const cardVariants = {
  elevated: {
    backgroundColor: 'white',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3
  },
  outlined: {
    backgroundColor: 'white',
    borderWidth: 1,
    borderColor: colors.neutral[200]
  },
  filled: {
    backgroundColor: colors.neutral[50],
    borderWidth: 0
  }
};

const cardPadding = {
  none: { padding: 0 },
  small: { padding: 8 },
  medium: { padding: 16 },
  large: { padding: 24 }
};
```

### Input Component

```typescript
interface InputProps {
  label: string;
  placeholder?: string;
  value: string;
  onChangeText: (text: string) => void;
  type: 'text' | 'email' | 'phone' | 'number' | 'password';
  error?: string;
  helperText?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

const inputTypes = {
  text: {
    keyboardType: 'default',
    secureTextEntry: false
  },
  email: {
    keyboardType: 'email-address',
    secureTextEntry: false
  },
  phone: {
    keyboardType: 'phone-pad',
    secureTextEntry: false
  },
  number: {
    keyboardType: 'numeric',
    secureTextEntry: false
  },
  password: {
    keyboardType: 'default',
    secureTextEntry: true
  }
};
```

### Equipment Card Component

```typescript
interface EquipmentCardProps {
  equipment: Equipment;
  variant: 'compact' | 'detailed' | 'list';
  onPress: () => void;
  showPrice?: boolean;
  showAvailability?: boolean;
  showRating?: boolean;
}

const equipmentCardVariants = {
  compact: {
    height: 120,
    imageHeight: 80,
    showSpecs: false,
    showDescription: false
  },
  detailed: {
    height: 200,
    imageHeight: 120,
    showSpecs: true,
    showDescription: true
  },
  list: {
    height: 100,
    imageHeight: 60,
    showSpecs: false,
    showDescription: false,
    horizontal: true
  }
};
```

### Order Card Component

```typescript
interface OrderCardProps {
  order: Order;
  variant: 'pending' | 'active' | 'completed';
  showActions?: boolean;
  onAction?: (action: string) => void;
}

const orderCardVariants = {
  pending: {
    backgroundColor: colors.warning + '10',
    borderColor: colors.warning,
    statusColor: colors.warning
  },
  active: {
    backgroundColor: colors.success + '10',
    borderColor: colors.success,
    statusColor: colors.success
  },
  completed: {
    backgroundColor: colors.neutral[50],
    borderColor: colors.neutral[200],
    statusColor: colors.neutral[500]
  }
};
```

## Layout System

### Spacing Scale

```typescript
export const spacing = {
  xs: 4,
  sm: 8,
  md: 16,
  lg: 24,
  xl: 32,
  xxl: 48,
  xxxl: 64
};
```

### Grid System

```typescript
export const grid = {
  columns: 12,
  gutter: 16,
  breakpoints: {
    xs: 0,
    sm: 576,
    md: 768,
    lg: 992,
    xl: 1200
  }
};
```

### Container Sizes

```typescript
export const containers = {
  sm: 540,
  md: 720,
  lg: 960,
  xl: 1140,
  xxl: 1320
};
```

## Animation System

### Transition Timing

```typescript
export const transitions = {
  fast: 150,
  normal: 300,
  slow: 500
};
```

### Easing Functions

```typescript
export const easing = {
  linear: 'linear',
  easeIn: 'ease-in',
  easeOut: 'ease-out',
  easeInOut: 'ease-in-out',
  spring: 'cubic-bezier(0.68, -0.55, 0.265, 1.55)'
};
```

### Animation Presets

```typescript
export const animations = {
  fadeIn: {
    from: { opacity: 0 },
    to: { opacity: 1 },
    duration: 300
  },
  slideUp: {
    from: { transform: [{ translateY: 20 }], opacity: 0 },
    to: { transform: [{ translateY: 0 }], opacity: 1 },
    duration: 300
  },
  scaleIn: {
    from: { transform: [{ scale: 0.9 }], opacity: 0 },
    to: { transform: [{ scale: 1 }], opacity: 1 },
    duration: 200
  }
};
```

## Icon System

### Icon Library

```typescript
export const icons = {
  // Navigation
  home: 'home',
  search: 'search',
  orders: 'list',
  wallet: 'wallet',
  profile: 'user',
  
  // Equipment
  excavator: 'truck',
  crane: 'arrow-up',
  bulldozer: 'square',
  loader: 'loader',
  
  // Actions
  add: 'plus',
  edit: 'edit',
  delete: 'trash',
  share: 'share',
  favorite: 'heart',
  
  // Status
  available: 'check-circle',
  busy: 'clock',
  maintenance: 'wrench',
  unavailable: 'x-circle',
  
  // Communication
  message: 'message-circle',
  phone: 'phone',
  email: 'mail',
  notification: 'bell'
};
```

### Icon Sizes

```typescript
export const iconSizes = {
  xs: 12,
  sm: 16,
  md: 20,
  lg: 24,
  xl: 32,
  xxl: 48
};
```

## Form Components

### Form Field

```typescript
interface FormFieldProps {
  label: string;
  required?: boolean;
  error?: string;
  helperText?: string;
  children: React.ReactNode;
}

const formFieldStyles = {
  label: {
    fontSize: 14,
    fontWeight: '500',
    color: colors.neutral[700],
    marginBottom: 4
  },
  error: {
    fontSize: 12,
    color: colors.error,
    marginTop: 4
  },
  helperText: {
    fontSize: 12,
    color: colors.neutral[500],
    marginTop: 4
  }
};
```

### Select Component

```typescript
interface SelectProps {
  options: { label: string; value: string }[];
  value: string;
  onValueChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
}

const selectStyles = {
  container: {
    borderWidth: 1,
    borderColor: colors.neutral[300],
    borderRadius: 8,
    paddingHorizontal: 12,
    paddingVertical: 8,
    backgroundColor: 'white'
  },
  selected: {
    color: colors.neutral[900]
  },
  placeholder: {
    color: colors.neutral[500]
  }
};
```

### Date Picker

```typescript
interface DatePickerProps {
  value: Date;
  onDateChange: (date: Date) => void;
  minimumDate?: Date;
  maximumDate?: Date;
  mode: 'date' | 'time' | 'datetime';
}

const datePickerStyles = {
  container: {
    borderWidth: 1,
    borderColor: colors.neutral[300],
    borderRadius: 8,
    paddingHorizontal: 12,
    paddingVertical: 8,
    backgroundColor: 'white'
  },
  text: {
    fontSize: 16,
    color: colors.neutral[900]
  }
};
```

## Navigation Components

### Tab Bar

```typescript
interface TabBarProps {
  tabs: Tab[];
  activeTab: string;
  onTabPress: (tabId: string) => void;
}

interface Tab {
  id: string;
  label: string;
  icon: string;
  badge?: number;
}

const tabBarStyles = {
  container: {
    backgroundColor: 'white',
    borderTopWidth: 1,
    borderTopColor: colors.neutral[200],
    paddingBottom: 8
  },
  tab: {
    flex: 1,
    alignItems: 'center',
    paddingVertical: 8
  },
  activeTab: {
    color: colors.primary[500]
  },
  inactiveTab: {
    color: colors.neutral[500]
  }
};
```

### Bottom Sheet

```typescript
interface BottomSheetProps {
  visible: boolean;
  onClose: () => void;
  children: React.ReactNode;
  snapPoints: number[];
}

const bottomSheetStyles = {
  container: {
    backgroundColor: 'white',
    borderTopLeftRadius: 16,
    borderTopRightRadius: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 8
  },
  handle: {
    width: 40,
    height: 4,
    backgroundColor: colors.neutral[300],
    borderRadius: 2,
    alignSelf: 'center',
    marginVertical: 8
  }
};
```

## Loading States

### Loading Spinner

```typescript
interface LoadingSpinnerProps {
  size: 'small' | 'medium' | 'large';
  color?: string;
}

const spinnerSizes = {
  small: 20,
  medium: 32,
  large: 48
};

const spinnerColors = {
  primary: colors.primary[500],
  secondary: colors.secondary[500],
  white: 'white'
};
```

### Skeleton Loader

```typescript
interface SkeletonProps {
  width: number | string;
  height: number | string;
  borderRadius?: number;
}

const skeletonStyles = {
  backgroundColor: colors.neutral[200],
  borderRadius: 4,
  opacity: 0.6
};
```

## Accessibility

### Accessibility Labels

```typescript
export const accessibilityLabels = {
  button: (label: string) => `Button: ${label}`,
  input: (label: string) => `Input field: ${label}`,
  card: (title: string) => `Card: ${title}`,
  tab: (label: string) => `Tab: ${label}`,
  image: (description: string) => `Image: ${description}`
};
```

### Focus Management

```typescript
export const focusStyles = {
  focusable: {
    focusable: true
  },
  focused: {
    borderColor: colors.primary[500],
    borderWidth: 2,
    outline: 'none'
  }
};
```

## Dark Mode Support

### Dark Mode Colors

```typescript
export const darkModeColors = {
  background: '#000000',
  surface: '#1a1a1a',
  text: '#ffffff',
  textSecondary: '#a0a0a0',
  border: '#333333',
  primary: colors.primary[400],
  secondary: colors.secondary[400]
};
```

### Theme Provider

```typescript
interface ThemeContextType {
  isDark: boolean;
  toggleTheme: () => void;
  colors: typeof colors;
}

export const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [isDark, setIsDark] = useState(false);
  
  const toggleTheme = () => setIsDark(!isDark);
  
  const themeColors = isDark ? darkModeColors : colors;
  
  return (
    <ThemeContext.Provider value={{ isDark, toggleTheme, colors: themeColors }}>
      {children}
    </ThemeContext.Provider>
  );
};
```

## Responsive Design

### Breakpoint Hooks

```typescript
export const useBreakpoint = () => {
  const [breakpoint, setBreakpoint] = useState('sm');
  
  useEffect(() => {
    const updateBreakpoint = () => {
      const width = window.innerWidth;
      if (width >= 1200) setBreakpoint('xl');
      else if (width >= 992) setBreakpoint('lg');
      else if (width >= 768) setBreakpoint('md');
      else if (width >= 576) setBreakpoint('sm');
      else setBreakpoint('xs');
    };
    
    updateBreakpoint();
    window.addEventListener('resize', updateBreakpoint);
    return () => window.removeEventListener('resize', updateBreakpoint);
  }, []);
  
  return breakpoint;
};
```

### Responsive Utilities

```typescript
export const responsive = {
  hide: (breakpoint: string) => ({
    [`@media (max-width: ${grid.breakpoints[breakpoint]}px)`]: {
      display: 'none'
    }
  }),
  show: (breakpoint: string) => ({
    [`@media (min-width: ${grid.breakpoints[breakpoint]}px)`]: {
      display: 'block'
    }
  })
};
```
