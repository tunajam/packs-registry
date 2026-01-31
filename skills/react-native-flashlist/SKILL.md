# FlashList for React Native

High-performance list rendering by Shopify.

## When to Apply

- Large lists (100+ items)
- Lists with complex items
- Infinite scrolling
- Lists with varying item heights
- Replacing FlatList for performance

## Installation

```bash
# Expo
npx expo install @shopify/flash-list

# React Native CLI
npm install @shopify/flash-list
cd ios && pod install
```

## Basic Usage

```tsx
import { FlashList } from '@shopify/flash-list';

function MyList() {
  return (
    <FlashList
      data={items}
      renderItem={({ item }) => <ListItem item={item} />}
      estimatedItemSize={100}
    />
  );
}
```

## Key Prop: estimatedItemSize

**Required for performance.** Estimate average item height:

```tsx
// Fixed height items - exact value
<FlashList
  estimatedItemSize={80}
  // ...
/>

// Variable heights - average
<FlashList
  estimatedItemSize={120} // Average of your items
  // ...
/>
```

### Finding the Right Value
```tsx
// During development, FlashList warns if estimate is off
// Adjust until warnings disappear
```

## Migration from FlatList

```tsx
// Before (FlatList)
import { FlatList } from 'react-native';

<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={(item) => item.id}
  getItemLayout={(data, index) => ({
    length: 80,
    offset: 80 * index,
    index,
  })}
/>

// After (FlashList)
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={renderItem}
  keyExtractor={(item) => item.id}
  estimatedItemSize={80}
/>
```

## Props Reference

```tsx
<FlashList
  // Required
  data={items}
  renderItem={({ item, index }) => <Item item={item} />}
  estimatedItemSize={100}
  
  // Layout
  horizontal={false}
  numColumns={2}
  inverted={false}
  
  // Performance
  drawDistance={250} // Pre-render distance
  
  // Headers/Footers
  ListHeaderComponent={<Header />}
  ListFooterComponent={<Footer />}
  ListEmptyComponent={<Empty />}
  
  // Separators
  ItemSeparatorComponent={() => <Divider />}
  
  // Callbacks
  onEndReached={() => loadMore()}
  onEndReachedThreshold={0.5}
  onViewableItemsChanged={handleViewable}
  
  // Keys
  keyExtractor={(item) => item.id}
  
  // Refresh
  refreshing={isRefreshing}
  onRefresh={handleRefresh}
  
  // Scroll
  showsVerticalScrollIndicator={true}
  contentContainerStyle={styles.content}
/>
```

## Variable Item Heights

```tsx
// FlashList handles variable heights automatically
// Just set a reasonable estimatedItemSize

function VariableHeightList() {
  return (
    <FlashList
      data={posts}
      renderItem={({ item }) => (
        <Post 
          text={item.text}
          image={item.image} // Some items have images
        />
      )}
      estimatedItemSize={200} // Average height
    />
  );
}
```

## Item Types (Heterogeneous Lists)

For lists with different item types:

```tsx
type ItemType = 'header' | 'item' | 'ad';

interface ListItem {
  type: ItemType;
  data: any;
}

function HeterogeneousList({ items }: { items: ListItem[] }) {
  const renderItem = ({ item }: { item: ListItem }) => {
    switch (item.type) {
      case 'header':
        return <SectionHeader data={item.data} />;
      case 'item':
        return <ListItem data={item.data} />;
      case 'ad':
        return <AdBanner data={item.data} />;
    }
  };

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      getItemType={(item) => item.type} // Important for recycling
      estimatedItemSize={80}
    />
  );
}
```

## Grid Layout

```tsx
<FlashList
  data={products}
  renderItem={({ item }) => <ProductCard item={item} />}
  numColumns={2}
  estimatedItemSize={200}
/>
```

### Dynamic Columns
```tsx
import { useWindowDimensions } from 'react-native';

function ResponsiveGrid() {
  const { width } = useWindowDimensions();
  const numColumns = Math.floor(width / 180); // 180px per column

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      numColumns={numColumns}
      estimatedItemSize={200}
      key={numColumns} // Re-render on column change
    />
  );
}
```

## Infinite Scroll

```tsx
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    const newItems = await fetchMoreItems(items.length);
    
    if (newItems.length === 0) {
      setHasMore(false);
    } else {
      setItems([...items, ...newItems]);
    }
    setLoading(false);
  };

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      estimatedItemSize={80}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        loading ? <ActivityIndicator /> : null
      }
    />
  );
}
```

## Pull to Refresh

```tsx
function RefreshableList() {
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = async () => {
    setRefreshing(true);
    await fetchLatestData();
    setRefreshing(false);
  };

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      estimatedItemSize={80}
      refreshing={refreshing}
      onRefresh={onRefresh}
    />
  );
}
```

## Optimized Item Component

```tsx
// ❌ Bad - creates new objects/functions each render
const renderItem = ({ item }) => (
  <TouchableOpacity
    style={{ padding: 16 }} // Inline style object
    onPress={() => handlePress(item.id)} // Inline function
  >
    <Text>{item.title}</Text>
  </TouchableOpacity>
);

// ✅ Good - memoized component with stable references
const ListItem = memo(({ item, onPress }) => (
  <TouchableOpacity style={styles.item} onPress={onPress}>
    <Text>{item.title}</Text>
  </TouchableOpacity>
));

function MyList() {
  const handlePress = useCallback((id) => {
    navigation.navigate('Detail', { id });
  }, [navigation]);

  const renderItem = useCallback(({ item }) => (
    <ListItem 
      item={item} 
      onPress={() => handlePress(item.id)} 
    />
  ), [handlePress]);

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      estimatedItemSize={80}
    />
  );
}
```

## Sticky Headers

```tsx
<FlashList
  data={itemsWithHeaders}
  renderItem={renderItem}
  getItemType={(item) => item.isHeader ? 'header' : 'item'}
  stickyHeaderIndices={headerIndices}
  estimatedItemSize={80}
/>
```

## Performance Tips

### 1. Measure Performance
```tsx
// FlashList logs performance warnings in dev mode
// Watch for:
// - "estimatedItemSize" warnings
// - Blank area warnings
// - Recycling issues
```

### 2. Avoid Expensive Renders
```tsx
// Use memo for list items
const Item = memo(({ data }) => (
  <View>
    <Text>{data.title}</Text>
  </View>
));
```

### 3. Optimize Images
```tsx
import { Image } from 'expo-image';

// Use optimized image libraries
<Image
  source={{ uri: item.thumbnail }}
  style={styles.image}
  contentFit="cover"
  transition={200}
/>
```

### 4. Increase Draw Distance
```tsx
// Pre-render more items if you have the memory
<FlashList
  drawDistance={500} // Default is 250
  // ...
/>
```

## Best Practices

1. **Always set estimatedItemSize** - Core performance metric
2. **Use getItemType** - For heterogeneous lists
3. **Memoize renderItem** - Prevent unnecessary re-renders
4. **Optimize item components** - Use memo, stable references
5. **Test on low-end devices** - Where performance matters most
6. **Check dev warnings** - FlashList tells you what's wrong
7. **Profile scroll performance** - 60fps is the goal
