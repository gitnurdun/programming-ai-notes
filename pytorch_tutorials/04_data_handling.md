# Data Handling in PyTorch - 30+ Questions and Answers

## 1. What is torch.utils.data.Dataset and how do you create a custom dataset?

**Answer:** `torch.utils.data.Dataset` is an abstract class representing a dataset. To create a custom dataset, you need to inherit from it and implement `__len__()` and `__getitem__()` methods:
```python
from torch.utils.data import Dataset

class CustomDataset(Dataset):
    def __init__(self, data, labels):
        self.data = data
        self.labels = labels
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        return self.data[idx], self.labels[idx]
```

## 2. What is torch.utils.data.DataLoader and what are its main parameters?

**Answer:** `DataLoader` is an iterator that provides batches of data from a dataset. Main parameters include:
- `dataset`: The dataset to load from
- `batch_size`: Number of samples per batch
- `shuffle`: Whether to shuffle data each epoch
- `num_workers`: Number of subprocesses for data loading
- `pin_memory`: Whether to use pinned memory for faster GPU transfer
- `drop_last`: Whether to drop the last incomplete batch

## 3. How do you create a custom dataset for image data?

**Answer:**
```python
import torch
from torch.utils.data import Dataset
from PIL import Image
import os

class ImageDataset(Dataset):
    def __init__(self, image_dir, labels_file, transform=None):
        self.image_dir = image_dir
        self.labels = self._load_labels(labels_file)
        self.transform = transform
        
    def __len__(self):
        return len(self.labels)
        
    def __getitem__(self, idx):
        image_path, label = self.labels[idx]
        image = Image.open(os.path.join(self.image_dir, image_path)).convert('RGB')
        
        if self.transform:
            image = self.transform(image)
            
        return image, label
        
    def _load_labels(self, labels_file):
        # Load labels from file
        with open(labels_file, 'r') as f:
            labels = [line.strip().split() for line in f]
        return labels
```

## 4. What are torchvision.transforms and how are they used?

**Answer:** `torchvision.transforms` provides common image transformations. They can be composed together using `transforms.Compose()`:
```python
from torchvision import transforms

transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Apply to dataset
dataset = CustomImageDataset(image_dir, labels_file, transform=transform)
```

## 5. How do you handle data augmentation in PyTorch?

**Answer:**
```python
train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomRotation(degrees=15),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    transforms.RandomResizedCrop(224, scale=(0.8, 1.0)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# For validation, use minimal transforms
val_transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])
```

## 6. How do you split a dataset into train, validation, and test sets?

**Answer:**
```python
from torch.utils.data import random_split

# Method 1: Using random_split
dataset = CustomDataset(data, labels)
train_size = int(0.8 * len(dataset))
val_size = int(0.1 * len(dataset))
test_size = len(dataset) - train_size - val_size

train_dataset, val_dataset, test_dataset = random_split(
    dataset, [train_size, val_size, test_size]
)

# Method 2: Using indices
indices = list(range(len(dataset)))
train_indices = indices[:train_size]
val_indices = indices[train_size:train_size+val_size]
test_indices = indices[train_size+val_size:]

train_dataset = torch.utils.data.Subset(dataset, train_indices)
val_dataset = torch.utils.data.Subset(dataset, val_indices)
test_dataset = torch.utils.data.Subset(dataset, test_indices)
```

## 7. What is the difference between random_split and manual splitting?

**Answer:** `random_split` randomly divides the dataset into specified sizes, ensuring no overlap between splits. Manual splitting using `Subset` with predefined indices gives you more control over which samples go to which split but requires careful index management. `random_split` is simpler for basic splits, while manual splitting is better for controlled experiments.

## 8. How do you handle imbalanced datasets?

**Answer:**
```python
from torch.utils.data import WeightedRandomSampler

# Calculate class weights
class_counts = [num_class_0, num_class_1, ...]
class_weights = [1.0 / count for count in class_counts]

# Create sample weights
sample_weights = [class_weights[label] for label in dataset.labels]

# Create weighted sampler
sampler = WeightedRandomSampler(sample_weights, len(sample_weights))

# Use with DataLoader
dataloader = DataLoader(dataset, batch_size=32, sampler=sampler)
```

## 9. How do you create a custom collate function for DataLoader?

**Answer:**
```python
def custom_collate_fn(batch):
    # batch is a list of (data, label) tuples
    data = [item[0] for item in batch]
    labels = [item[1] for item in batch]
    
    # Handle variable length sequences
    if isinstance(data[0], torch.Tensor) and data[0].dim() > 1:
        # Pad sequences to same length
        data = torch.nn.utils.rnn.pad_sequence(data, batch_first=True)
    else:
        data = torch.stack(data)
        
    labels = torch.tensor(labels)
    return data, labels

# Use with DataLoader
dataloader = DataLoader(dataset, batch_size=32, collate_fn=custom_collate_fn)
```

## 10. How do you handle variable-length sequences in datasets?

**Answer:**
```python
from torch.nn.utils.rnn import pad_sequence

class SequenceDataset(Dataset):
    def __init__(self, sequences, labels):
        self.sequences = sequences
        self.labels = labels
        
    def __len__(self):
        return len(self.sequences)
        
    def __getitem__(self, idx):
        return torch.tensor(self.sequences[idx]), self.labels[idx]

def collate_fn(batch):
    sequences = [item[0] for item in batch]
    labels = torch.tensor([item[1] for item in batch])
    
    # Pad sequences to the same length
    padded_sequences = pad_sequence(sequences, batch_first=True)
    
    return padded_sequences, labels

# Use with DataLoader
dataloader = DataLoader(dataset, batch_size=32, collate_fn=collate_fn)
```

## 11. How do you load data from files (CSV, JSON, etc.) into PyTorch datasets?

**Answer:**
```python
import pandas as pd
import json

class CSVDataset(Dataset):
    def __init__(self, csv_file, transform=None):
        self.data = pd.read_csv(csv_file)
        self.transform = transform
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        row = self.data.iloc[idx]
        features = torch.tensor(row[:-1].values, dtype=torch.float32)
        label = torch.tensor(row[-1], dtype=torch.long)
        
        if self.transform:
            features = self.transform(features)
            
        return features, label

class JSONDataset(Dataset):
    def __init__(self, json_file):
        with open(json_file, 'r') as f:
            self.data = json.load(f)
            
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        item = self.data[idx]
        features = torch.tensor(item['features'], dtype=torch.float32)
        label = torch.tensor(item['label'], dtype=torch.long)
        return features, label
```

## 12. How do you handle large datasets that don't fit in memory?

**Answer:**
```python
class LargeDataset(Dataset):
    def __init__(self, data_paths, labels):
        # Store only paths, not actual data
        self.data_paths = data_paths
        self.labels = labels
        
    def __len__(self):
        return len(self.data_paths)
        
    def __getitem__(self, idx):
        # Load data on-demand
        data_path = self.data_paths[idx]
        label = self.labels[idx]
        
        # Load data from file
        data = torch.load(data_path)  # or any other loading method
        
        return data, label

# Use with multiple workers for faster loading
dataloader = DataLoader(dataset, batch_size=32, num_workers=4, pin_memory=True)
```

## 13. What is the purpose of pin_memory in DataLoader?

**Answer:** `pin_memory=True` enables faster data transfer from CPU to GPU by using pinned (page-locked) memory. Pinned memory allows for asynchronous memory transfers, which can significantly speed up training when using GPU. This is especially beneficial when `num_workers > 0`.

## 14. How do you implement data preprocessing in a dataset class?

**Answer:**
```python
class PreprocessedDataset(Dataset):
    def __init__(self, data_file, preprocessor=None):
        self.data = self._load_data(data_file)
        self.preprocessor = preprocessor
        
    def _load_data(self, data_file):
        # Load raw data
        data = pd.read_csv(data_file)
        return data
        
    def _preprocess(self, sample):
        # Apply preprocessing transformations
        if self.preprocessor:
            sample = self.preprocessor.transform(sample.reshape(1, -1))[0]
        return sample
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        row = self.data.iloc[idx]
        features = row[:-1].values.astype(np.float32)
        label = row[-1]
        
        # Apply preprocessing
        features = self._preprocess(features)
        
        return torch.tensor(features), label
```

## 15. How do you create a dataset for text data?

**Answer:**
```python
class TextDataset(Dataset):
    def __init__(self, texts, labels, tokenizer, max_length=512):
        self.texts = texts
        self.labels = labels
        self.tokenizer = tokenizer
        self.max_length = max_length
        
    def __len__(self):
        return len(self.texts)
        
    def __getitem__(self, idx):
        text = str(self.texts[idx])
        label = self.labels[idx]
        
        # Tokenize text
        encoding = self.tokenizer(
            text,
            truncation=True,
            padding='max_length',
            max_length=self.max_length,
            return_tensors='pt'
        )
        
        return {
            'input_ids': encoding['input_ids'].flatten(),
            'attention_mask': encoding['attention_mask'].flatten(),
            'label': torch.tensor(label, dtype=torch.long)
        }
```

## 16. How do you handle multi-modal data in PyTorch?

**Answer:**
```python
class MultiModalDataset(Dataset):
    def __init__(self, image_paths, texts, labels, image_transform=None, tokenizer=None):
        self.image_paths = image_paths
        self.texts = texts
        self.labels = labels
        self.image_transform = image_transform
        self.tokenizer = tokenizer
        
    def __len__(self):
        return len(self.labels)
        
    def __getitem__(self, idx):
        # Load image
        image = Image.open(self.image_paths[idx]).convert('RGB')
        if self.image_transform:
            image = self.image_transform(image)
            
        # Process text
        text = str(self.texts[idx])
        if self.tokenizer:
            encoding = self.tokenizer(
                text, truncation=True, padding='max_length',
                max_length=128, return_tensors='pt'
            )
            text_data = {
                'input_ids': encoding['input_ids'].flatten(),
                'attention_mask': encoding['attention_mask'].flatten()
            }
        else:
            text_data = text
            
        label = self.labels[idx]
        
        return {
            'image': image,
            'text': text_data,
            'label': torch.tensor(label, dtype=torch.long)
        }
```

## 17. What is the difference between batch_size and batch_sampler?

**Answer:** `batch_size` is a simple parameter that determines how many samples per batch. `batch_sampler` is a more flexible approach that allows custom batch creation logic. When `batch_sampler` is specified, `batch_size`, `shuffle`, `sampler`, and `drop_last` are ignored:
```python
from torch.utils.data import BatchSampler, SequentialSampler

# Custom batch sampler
sampler = SequentialSampler(dataset)
batch_sampler = BatchSampler(sampler, batch_size=32, drop_last=True)
dataloader = DataLoader(dataset, batch_sampler=batch_sampler)
```

## 18. How do you implement a custom sampler?

**Answer:**
```python
from torch.utils.data import Sampler

class CustomSampler(Sampler):
    def __init__(self, data_source, sampling_weights):
        self.data_source = data_source
        self.sampling_weights = sampling_weights
        
    def __iter__(self):
        # Custom sampling logic
        indices = torch.multinomial(
            self.sampling_weights, len(self.data_source), 
            replacement=True
        )
        return iter(indices.tolist())
        
    def __len__(self):
        return len(self.data_source)

# Use with DataLoader
sampler = CustomSampler(dataset, sampling_weights)
dataloader = DataLoader(dataset, batch_size=32, sampler=sampler)
```

## 19. How do you handle data loading errors gracefully?

**Answer:**
```python
class RobustDataset(Dataset):
    def __init__(self, data_paths, labels):
        self.data_paths = data_paths
        self.labels = labels
        self.error_count = 0
        
    def __len__(self):
        return len(self.data_paths)
        
    def __getitem__(self, idx):
        try:
            data_path = self.data_paths[idx]
            label = self.labels[idx]
            data = self._load_data(data_path)
            return data, label
        except Exception as e:
            self.error_count += 1
            print(f"Error loading data at index {idx}: {e}")
            # Return a default/fallback sample or skip
            return self.__getitem__((idx + 1) % len(self))
            
    def _load_data(self, data_path):
        # Your data loading logic here
        return torch.load(data_path)
```

## 20. How do you use prefetching to speed up data loading?

**Answer:** Prefetching is automatically handled when using `num_workers > 0` in DataLoader. For additional control, you can implement prefetching manually:
```python
# Using multiple workers (built-in prefetching)
dataloader = DataLoader(
    dataset, 
    batch_size=32, 
    num_workers=4,  # Prefetches data in background
    pin_memory=True
)

# Manual prefetching with iterator
class PrefetchGenerator:
    def __init__(self, generator, max_prefetch=1):
        self.generator = generator
        self.max_prefetch = max_prefetch
        
    def __iter__(self):
        # Implementation for manual prefetching
        pass
```

## 21. How do you implement data caching to avoid repeated preprocessing?

**Answer:**
```python
import pickle
import os

class CachedDataset(Dataset):
    def __init__(self, data_source, cache_dir, transform=None):
        self.data_source = data_source
        self.cache_dir = cache_dir
        self.transform = transform
        os.makedirs(cache_dir, exist_ok=True)
        
    def __len__(self):
        return len(self.data_source)
        
    def __getitem__(self, idx):
        cache_path = os.path.join(self.cache_dir, f"item_{idx}.pkl")
        
        # Check if cached
        if os.path.exists(cache_path):
            with open(cache_path, 'rb') as f:
                item = pickle.load(f)
        else:
            # Process and cache
            item = self._process_item(idx)
            with open(cache_path, 'wb') as f:
                pickle.dump(item, f)
                
        return item
        
    def _process_item(self, idx):
        # Expensive preprocessing operations
        raw_data = self.data_source[idx]
        processed_data = self.transform(raw_data) if self.transform else raw_data
        return processed_data
```

## 22. How do you handle distributed data loading in PyTorch?

**Answer:**
```python
from torch.utils.data.distributed import DistributedSampler

# In distributed training setup
def setup_dataloader(dataset, batch_size, world_size, rank):
    sampler = DistributedSampler(
        dataset,
        num_replicas=world_size,
        rank=rank,
        shuffle=True
    )
    
    dataloader = DataLoader(
        dataset,
        batch_size=batch_size,
        sampler=sampler,
        num_workers=4
    )
    
    return dataloader

# Usage in distributed training
# sampler.set_epoch(epoch)  # Shuffle differently each epoch
```

## 23. How do you implement data validation during loading?

**Answer:**
```python
class ValidatedDataset(Dataset):
    def __init__(self, data_paths, labels, validator=None):
        self.data_paths = data_paths
        self.labels = labels
        self.validator = validator
        self.valid_indices = self._validate_data()
        
    def _validate_data(self):
        valid_indices = []
        for i, (path, label) in enumerate(zip(self.data_paths, self.labels)):
            try:
                data = self._load_item(path)
                if self.validator is None or self.validator(data, label):
                    valid_indices.append(i)
            except Exception as e:
                print(f"Invalid data at index {i}: {e}")
        return valid_indices
        
    def __len__(self):
        return len(self.valid_indices)
        
    def __getitem__(self, idx):
        actual_idx = self.valid_indices[idx]
        data_path = self.data_paths[actual_idx]
        label = self.labels[actual_idx]
        data = self._load_item(data_path)
        return data, label
        
    def _load_item(self, path):
        # Your loading logic
        return torch.load(path)
```

## 24. How do you handle time-series data in PyTorch datasets?

**Answer:**
```python
class TimeSeriesDataset(Dataset):
    def __init__(self, data, sequence_length, prediction_horizon=1):
        self.data = data
        self.sequence_length = sequence_length
        self.prediction_horizon = prediction_horizon
        
    def __len__(self):
        return len(self.data) - self.sequence_length - self.prediction_horizon + 1
        
    def __getitem__(self, idx):
        # Extract sequence
        sequence = self.data[idx:idx + self.sequence_length]
        # Extract target (next value or sequence)
        target = self.data[idx + self.sequence_length:
                          idx + self.sequence_length + self.prediction_horizon]
        
        return (
            torch.tensor(sequence, dtype=torch.float32),
            torch.tensor(target, dtype=torch.float32)
        )
```

## 25. How do you implement data shuffling with reproducible results?

**Answer:**
```python
import random

class ReproducibleDataset(Dataset):
    def __init__(self, data, labels, seed=42):
        self.data = data
        self.labels = labels
        self.seed = seed
        self.indices = list(range(len(data)))
        self._shuffle_indices()
        
    def _shuffle_indices(self):
        # Use seed for reproducible shuffling
        random_state = random.getstate()
        random.seed(self.seed)
        random.shuffle(self.indices)
        random.setstate(random_state)
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        actual_idx = self.indices[idx]
        return self.data[actual_idx], self.labels[actual_idx]
        
    def set_epoch(self, epoch):
        # Different shuffle for each epoch
        random_state = random.getstate()
        random.seed(self.seed + epoch)
        random.shuffle(self.indices)
        random.setstate(random_state)
```

## 26. How do you handle data loading for reinforcement learning environments?

**Answer:**
```python
class RLDataset(Dataset):
    def __init__(self, env, num_episodes, max_steps_per_episode=1000):
        self.data = []
        self._collect_data(env, num_episodes, max_steps_per_episode)
        
    def _collect_data(self, env, num_episodes, max_steps):
        for episode in range(num_episodes):
            state = env.reset()
            episode_data = []
            
            for step in range(max_steps):
                # Random action or use policy
                action = env.action_space.sample()
                next_state, reward, done, _ = env.step(action)
                
                # Store transition
                self.data.append({
                    'state': state,
                    'action': action,
                    'reward': reward,
                    'next_state': next_state,
                    'done': done
                })
                
                if done:
                    break
                state = next_state
                
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        transition = self.data[idx]
        return (
            torch.tensor(transition['state'], dtype=torch.float32),
            torch.tensor(transition['action']),
            torch.tensor(transition['reward'], dtype=torch.float32),
            torch.tensor(transition['next_state'], dtype=torch.float32),
            torch.tensor(transition['done'], dtype=torch.bool)
        )
```

## 27. How do you implement data versioning for datasets?

**Answer:**
```python
import hashlib

class VersionedDataset(Dataset):
    def __init__(self, data_source, version="1.0"):
        self.data_source = data_source
        self.version = version
        self.data_hash = self._compute_hash()
        self.data = self._load_versioned_data()
        
    def _compute_hash(self):
        # Compute hash of data source for versioning
        if isinstance(self.data_source, str):
            with open(self.data_source, 'rb') as f:
                return hashlib.md5(f.read()).hexdigest()
        else:
            # For other data sources
            return hashlib.md5(str(self.data_source).encode()).hexdigest()
            
    def _load_versioned_data(self):
        cache_path = f"cache_{self.version}_{self.data_hash[:8]}.pkl"
        
        if os.path.exists(cache_path):
            with open(cache_path, 'rb') as f:
                return pickle.load(f)
        else:
            # Process and cache
            data = self._process_data()
            with open(cache_path, 'wb') as f:
                pickle.dump(data, f)
            return data
            
    def _process_data(self):
        # Your data processing logic
        pass
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        return self.data[idx]
```

## 28. How do you handle data loading with memory mapping for large datasets?

**Answer:**
```python
import numpy as np

class MemoryMappedDataset(Dataset):
    def __init__(self, data_file, labels_file):
        # Memory map large arrays
        self.data = np.load(data_file, mmap_mode='r')
        self.labels = np.load(labels_file, mmap_mode='r')
        
    def __len__(self):
        return len(self.data)
        
    def __getitem__(self, idx):
        # Data is loaded on-demand from disk
        data = torch.from_numpy(self.data[idx]).float()
        label = torch.from_numpy(self.labels[idx]).long()
        return data, label
        
# Usage with DataLoader
# Note: num_workers should be 0 or data will be copied to each worker
dataset = MemoryMappedDataset('large_data.npy', 'labels.npy')
dataloader = DataLoader(dataset, batch_size=32, num_workers=0)
```

## 29. How do you implement a dataset with online data generation?

**Answer:**
```python
class OnlineGeneratedDataset(Dataset):
    def __init__(self, generator_func, num_samples, seed=42):
        self.generator_func = generator_func
        self.num_samples = num_samples
        self.seed = seed
        self.rng = np.random.default_rng(seed)
        
    def __len__(self):
        return self.num_samples
        
    def __getitem__(self, idx):
        # Generate data on-the-fly
        # Set seed for reproducibility
        with np.random.seed(self.seed + idx):
            data, label = self.generator_func()
        return torch.tensor(data), torch.tensor(label)
        
# Example generator function
def generate_synthetic_data():
    # Generate synthetic sample
    data = np.random.randn(10)  # 10 features
    label = np.random.randint(0, 2)  # Binary label
    return data, label
    
# Usage
dataset = OnlineGeneratedDataset(generate_synthetic_data, num_samples=10000)
```

## 30. How do you handle data loading for graph neural networks?

**Answer:**
```python
from torch_geometric.data import Data

class GraphDataset(Dataset):
    def __init__(self, graph_files):
        self.graph_files = graph_files
        
    def __len__(self):
        return len(self.graph_files)
        
    def __getitem__(self, idx):
        # Load graph data
        graph_data = torch.load(self.graph_files[idx])
        
        # Convert to PyG Data object
        data = Data(
            x=graph_data['node_features'],      # Node features
            edge_index=graph_data['edge_index'], # Edge connections
            edge_attr=graph_data['edge_features'], # Edge features (optional)
            y=graph_data['graph_label'],        # Graph label
            num_nodes=graph_data['num_nodes']
        )
        
        return data

# For batch processing with PyG
from torch_geometric.loader import DataLoader

dataset = GraphDataset(graph_files)
dataloader = DataLoader(dataset, batch_size=32, shuffle=True)
```

## 31. How do you implement a dataset with progressive resizing for images?

**Answer:**
```python
class ProgressiveResizeDataset(Dataset):
    def __init__(self, image_paths, labels, sizes=[128, 224, 299]):
        self.image_paths = image_paths
        self.labels = labels
        self.sizes = sizes
        self.current_size_idx = 0
        
    def set_size(self, size_idx):
        """Change the current image size during training"""
        self.current_size_idx = size_idx
        
    def __len__(self):
        return len(self.image_paths)
        
    def __getitem__(self, idx):
        image = Image.open(self.image_paths[idx]).convert('RGB')
        label = self.labels[idx]
        
        # Resize to current size
        current_size = self.sizes[self.current_size_idx]
        transform = transforms.Compose([
            transforms.Resize((current_size, current_size)),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                               std=[0.229, 0.224, 0.225])
        ])
        
        image = transform(image)
        return image, torch.tensor(label, dtype=torch.long)
        
# Usage in training loop
# dataset.set_size(epoch // 10)  # Increase size every 10 epochs
```