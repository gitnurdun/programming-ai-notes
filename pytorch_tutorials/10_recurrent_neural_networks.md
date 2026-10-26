# Recurrent Neural Networks (RNNs) in PyTorch - 30+ Questions and Answers

## 1. What are Recurrent Neural Networks (RNNs) and how do they work?

**Answer:** Recurrent Neural Networks (RNNs) are a class of neural networks designed to handle sequential data by maintaining an internal state (memory) that captures information about past inputs. Unlike feedforward networks, RNNs have connections that form directed cycles, allowing them to exhibit dynamic temporal behavior. The key concept is that the network processes sequences element by element, with each step taking into account both the current input and the hidden state from the previous step.

The basic RNN formula: h_t = tanh(W_hh * h_{t-1} + W_xh * x_t + b_h), where:
- h_t is the hidden state at time t
- x_t is the input at time t
- W_hh, W_xh are weight matrices
- b_h is the bias term

## 2. How do you implement a basic RNN in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BasicRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1):
        super(BasicRNN, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # RNN layer
        self.rnn = nn.RNN(input_size, hidden_size, num_layers, batch_first=True)
        
        # Output layer
        self.fc = nn.Linear(hidden_size, output_size)
        
    def forward(self, x, hidden=None):
        # x shape: (batch_size, seq_len, input_size)
        batch_size = x.size(0)
        
        # Initialize hidden state if not provided
        if hidden is None:
            hidden = torch.zeros(self.num_layers, batch_size, self.hidden_size)
            hidden = hidden.to(x.device)
        
        # Forward pass through RNN
        rnn_out, hidden = self.rnn(x, hidden)
        
        # Apply linear layer to all time steps
        output = self.fc(rnn_out)  # (batch_size, seq_len, output_size)
        
        return output, hidden

# Example usage
input_size = 10
hidden_size = 20
output_size = 5
seq_len = 15
batch_size = 32

model = BasicRNN(input_size, hidden_size, output_size)
input_tensor = torch.randn(batch_size, seq_len, input_size)
output, hidden = model(input_tensor)
print(f"Input shape: {input_tensor.shape}")
print(f"Output shape: {output.shape}")
print(f"Hidden shape: {hidden.shape}")
```

## 3. What is the difference between RNN, LSTM, and GRU?

**Answer:**
- **RNN (Basic Recurrent Neural Network):**
  - Simple recurrent connections
  - Prone to vanishing gradient problems
  - Limited ability to capture long-term dependencies
  - Formula: h_t = tanh(W_hh * h_{t-1} + W_xh * x_t + b_h)

- **LSTM (Long Short-Term Memory):**
  - Has memory cells and three gates (input, forget, output)
  - Better at capturing long-term dependencies
  - Uses cell state to maintain information over time
  - More complex but very effective for long sequences

- **GRU (Gated Recurrent Unit):**
  - Simpler than LSTM with two gates (reset, update)
  - Combines cell state and hidden state
  - Faster to train than LSTM
  - Often performs similarly to LSTM with fewer parameters

## 4. How do you implement LSTM in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn

class BasicLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1, dropout=0.0):
        super(BasicLSTM, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # LSTM layer
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, 
                           batch_first=True, dropout=dropout)
        
        # Output layer
        self.fc = nn.Linear(hidden_size, output_size)
        
    def forward(self, x, hidden=None):
        batch_size = x.size(0)
        
        # Initialize hidden states if not provided
        if hidden is None:
            h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
            c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
            hidden = (h0, c0)
        
        # Forward pass through LSTM
        lstm_out, hidden = self.lstm(x, hidden)
        
        # Apply linear layer to all time steps
        output = self.fc(lstm_out)
        
        return output, hidden

class LSTMForClassification(nn.Module):
    def __init__(self, input_size, hidden_size, num_classes, num_layers=1):
        super(LSTMForClassification, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, num_classes)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Initialize hidden states
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # Forward pass
        lstm_out, (hn, cn) = self.lstm(x, (h0, c0))
        
        # Use the last time step for classification
        output = self.fc(self.dropout(lstm_out[:, -1, :]))
        
        return output

# Example usage
lstm_model = LSTMForClassification(input_size=50, hidden_size=128, num_classes=10)
input_seq = torch.randn(32, 20, 50)  # (batch, seq_len, input_size)
output = lstm_model(input_seq)
print(f"LSTM output shape: {output.shape}")
```

## 5. How do you implement GRU in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn

class BasicGRU(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1, dropout=0.0):
        super(BasicGRU, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # GRU layer
        self.gru = nn.GRU(input_size, hidden_size, num_layers, 
                         batch_first=True, dropout=dropout)
        
        # Output layer
        self.fc = nn.Linear(hidden_size, output_size)
        
    def forward(self, x, hidden=None):
        batch_size = x.size(0)
        
        # Initialize hidden state if not provided
        if hidden is None:
            hidden = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # Forward pass through GRU
        gru_out, hidden = self.gru(x, hidden)
        
        # Apply linear layer to all time steps
        output = self.fc(gru_out)
        
        return output, hidden

class GRUForSequenceClassification(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_classes, num_layers=1):
        super(GRUForSequenceClassification, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # Embedding layer for sequences
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # GRU layer
        self.gru = nn.GRU(embedding_dim, hidden_size, num_layers, 
                         batch_first=True, dropout=0.3)
        
        # Output layers
        self.fc1 = nn.Linear(hidden_size, hidden_size // 2)
        self.fc2 = nn.Linear(hidden_size // 2, num_classes)
        self.dropout = nn.Dropout(0.5)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Embedding
        embedded = self.embedding(x)  # (batch, seq_len, embedding_dim)
        
        # Initialize hidden state
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # GRU forward pass
        gru_out, hidden = self.gru(embedded, h0)
        
        # Use the last time step
        last_hidden = gru_out[:, -1, :]  # (batch, hidden_size)
        
        # Classification layers
        out = self.dropout(last_hidden)
        out = self.fc1(out)
        out = self.relu(out)
        out = self.dropout(out)
        out = self.fc2(out)
        
        return out

# Example usage
gru_model = GRUForSequenceClassification(
    vocab_size=10000, 
    embedding_dim=100, 
    hidden_size=128, 
    num_classes=5, 
    num_layers=2
)
input_seq = torch.randint(0, 10000, (32, 50))  # (batch, seq_len) with integer indices
output = gru_model(input_seq)
print(f"GRU output shape: {output.shape}")
```

## 6. How do you handle variable-length sequences in RNNs?

**Answer:**
```python
import torch
import torch.nn as nn
from torch.nn.utils.rnn import pad_sequence, pack_padded_sequence, pad_packed_sequence

class PaddedRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1):
        super(PaddedRNN, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # LSTM for better gradient flow
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, 
                           batch_first=True, dropout=0.3)
        self.fc = nn.Linear(hidden_size, output_size)
        
    def forward(self, x, lengths):
        # Pack padded sequences
        packed_input = pack_padded_sequence(x, lengths, batch_first=True, enforce_sorted=False)
        
        # LSTM forward pass
        packed_output, (hidden, cell) = self.lstm(packed_input)
        
        # Unpack sequences
        output, output_lengths = pad_packed_sequence(packed_output, batch_first=True)
        
        # Apply linear layer
        output = self.fc(output)
        
        return output, output_lengths

def collate_fn(batch):
    """Collate function for variable length sequences"""
    # Sort batch by sequence length (descending) for pack_padded_sequence
    batch.sort(key=lambda x: len(x[0]), reverse=True)
    
    sequences, labels = zip(*batch)
    
    # Get lengths
    lengths = [len(seq) for seq in sequences]
    
    # Pad sequences
    padded_sequences = pad_sequence(sequences, batch_first=True, padding_value=0)
    
    return padded_sequences, torch.tensor(labels), torch.tensor(lengths)

# Example with different sequence lengths
def create_variable_length_data():
    # Create sample sequences of different lengths
    seq1 = torch.randn(10, 50)  # Length 10
    seq2 = torch.randn(15, 50)  # Length 15
    seq3 = torch.randn(8, 50)   # Length 8
    
    sequences = [seq1, seq2, seq3]
    labels = torch.tensor([0, 1, 2])
    
    # Pad sequences
    padded = pad_sequence(sequences, batch_first=True, padding_value=0)
    lengths = torch.tensor([len(seq) for seq in sequences])
    
    return padded, labels, lengths

# Usage
padded_data, labels, lengths = create_variable_length_data()
print(f"Padded shape: {padded_data.shape}")
print(f"Lengths: {lengths}")

# Model with packed sequences
model = PaddedRNN(input_size=50, hidden_size=64, output_size=10)
output, out_lengths = model(padded_data, lengths)
print(f"Output shape: {output.shape}")
print(f"Output lengths: {out_lengths}")
```

## 7. How do you implement bidirectional RNNs?

**Answer:**
```python
import torch
import torch.nn as nn

class BidirectionalLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1):
        super(BidirectionalLSTM, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # Bidirectional LSTM
        self.lstm = nn.LSTM(
            input_size, 
            hidden_size, 
            num_layers, 
            batch_first=True,
            bidirectional=True  # This makes it bidirectional
        )
        
        # The output size is doubled because of concatenation of forward and backward
        self.fc = nn.Linear(hidden_size * 2, output_size)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Initialize hidden states for both directions
        h0 = torch.zeros(self.num_layers * 2, batch_size, self.hidden_size).to(x.device)  # *2 for bidirectional
        c0 = torch.zeros(self.num_layers * 2, batch_size, self.hidden_size).to(x.device)
        
        # Forward pass
        lstm_out, (hn, cn) = self.lstm(x, (h0, c0))
        
        # Apply dropout and linear layer
        lstm_out = self.dropout(lstm_out)
        output = self.fc(lstm_out)
        
        return output, (hn, cn)

class BidirectionalRNN(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_classes, num_layers=1):
        super(BidirectionalRNN, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # Bidirectional RNN
        self.rnn = nn.LSTM(
            embedding_dim,
            hidden_size,
            num_layers,
            batch_first=True,
            bidirectional=True,
            dropout=0.3
        )
        
        # Linear layer (hidden_size * 2 for bidirectional)
        self.fc = nn.Linear(hidden_size * 2, num_classes)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        embedded = self.embedding(x)
        
        # Initialize hidden state
        batch_size = x.size(0)
        h0 = torch.zeros(self.num_layers * 2, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers * 2, batch_size, self.hidden_size).to(x.device)
        
        # Forward pass
        output, (hidden, cell) = self.rnn(embedded, (h0, c0))
        
        # Use the last time step's output (concatenated forward and backward)
        last_output = output[:, -1, :]  # Shape: (batch, hidden_size * 2)
        
        # Classification
        out = self.dropout(last_output)
        out = self.fc(out)
        
        return out

# Example usage
bidirectional_model = BidirectionalRNN(
    vocab_size=10000,
    embedding_dim=128,
    hidden_size=64,
    num_classes=5,
    num_layers=2
)

# Test with sample data
sample_input = torch.randint(0, 10000, (16, 50))  # (batch, seq_len)
output = bidirectional_model(sample_input)
print(f"Bidirectional RNN output shape: {output.shape}")

# Compare forward vs bidirectional
class ForwardOnlyRNN(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_classes, num_layers=1):
        super(ForwardOnlyRNN, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.rnn = nn.LSTM(embedding_dim, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, num_classes)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        embedded = self.embedding(x)
        output, _ = self.rnn(embedded)
        last_output = output[:, -1, :]  # Shape: (batch, hidden_size)
        output = self.dropout(last_output)
        output = self.fc(output)
        return output

forward_model = ForwardOnlyRNN(10000, 128, 64, 5, 2)
forward_output = forward_model(sample_input)
print(f"Forward-only RNN output shape: {forward_output.shape}")

print(f"Bidirectional has {bidirectional_model.fc.in_features} inputs vs {forward_model.fc.in_features} for forward-only")
```

## 8. How do you implement attention mechanisms with RNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class Attention(nn.Module):
    def __init__(self, hidden_size):
        super(Attention, self).__init__()
        self.hidden_size = hidden_size
        self.attn = nn.Linear(self.hidden_size * 2, hidden_size)
        self.v = nn.Parameter(torch.rand(hidden_size))
        self.softmax = nn.Softmax(dim=1)
    
    def forward(self, hidden, encoder_outputs):
        # hidden: (batch_size, hidden_size)
        # encoder_outputs: (batch_size, seq_len, hidden_size)
        batch_size = encoder_outputs.size(0)
        seq_len = encoder_outputs.size(1)
        
        # Repeat hidden state for each time step
        hidden = hidden.unsqueeze(1).repeat(1, seq_len, 1)
        
        # Concatenate hidden and encoder outputs
        combined = torch.cat((hidden, encoder_outputs), dim=2)
        
        # Calculate attention energies
        energies = torch.tanh(self.attn(combined))
        energies = energies.transpose(1, 2)  # (batch, hidden_size, seq_len)
        
        # Calculate attention weights
        v = self.v.repeat(batch_size, 1).unsqueeze(1)  # (batch, 1, hidden_size)
        energies = torch.bmm(v, energies)  # (batch, 1, seq_len)
        attn_weights = self.softmax(energies.squeeze(1))  # (batch, seq_len)
        
        # Apply attention weights to encoder outputs
        attn_applied = torch.bmm(attn_weights.unsqueeze(1), encoder_outputs)
        
        return attn_applied.squeeze(1), attn_weights

class RNNWithAttention(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, output_size, num_layers=1):
        super(RNNWithAttention, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, 
                           batch_first=True, dropout=0.3)
        self.attention = Attention(hidden_size)
        self.fc = nn.Linear(hidden_size * 2, output_size)  # *2 for context vector + hidden
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        embedded = self.embedding(x)
        
        # LSTM forward pass
        lstm_out, (hidden, cell) = self.lstm(embedded)
        
        # Take the last hidden state
        last_hidden = hidden[-1]  # (batch, hidden_size)
        
        # Apply attention mechanism
        context, attn_weights = self.attention(last_hidden, lstm_out)
        
        # Combine context and last hidden state
        combined = torch.cat((context, last_hidden), dim=1)
        
        # Apply dropout and linear layer
        output = self.dropout(combined)
        output = self.fc(output)
        
        return output, attn_weights

# Multi-head attention for RNNs
class MultiHeadAttention(nn.Module):
    def __init__(self, hidden_size, num_heads=8):
        super(MultiHeadAttention, self).__init__()
        assert hidden_size % num_heads == 0
        
        self.hidden_size = hidden_size
        self.num_heads = num_heads
        self.head_dim = hidden_size // num_heads
        
        self.query = nn.Linear(hidden_size, hidden_size)
        self.key = nn.Linear(hidden_size, hidden_size)
        self.value = nn.Linear(hidden_size, hidden_size)
        self.fc_out = nn.Linear(hidden_size, hidden_size)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.shape[0]
        
        # Linear transformations
        Q = self.query(query)
        K = self.key(key)
        V = self.value(value)
        
        # Reshape for multi-head attention
        Q = Q.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        
        # Scaled dot-product attention
        energy = torch.matmul(Q, K.transpose(-2, -1)) / (self.head_dim ** 0.5)
        
        if mask is not None:
            energy = energy.masked_fill(mask == 0, float("-1e20"))
        
        attention = torch.softmax(energy, dim=-1)
        out = torch.matmul(attention, V)
        
        # Combine heads
        out = out.transpose(1, 2).contiguous().view(batch_size, -1, self.hidden_size)
        out = self.fc_out(out)
        
        return out, attention

class RNNWithMultiHeadAttention(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, output_size, num_layers=1, num_heads=8):
        super(RNNWithMultiHeadAttention, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, 
                           batch_first=True, dropout=0.3)
        self.attention = MultiHeadAttention(hidden_size, num_heads)
        self.fc = nn.Linear(hidden_size, output_size)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        embedded = self.embedding(x)
        
        # LSTM forward pass
        lstm_out, (hidden, cell) = self.lstm(embedded)
        
        # Apply multi-head attention
        attended_out, attention_weights = self.attention(lstm_out, lstm_out, lstm_out)
        
        # Use the last time step for classification
        last_output = attended_out[:, -1, :]
        
        # Apply dropout and linear layer
        output = self.dropout(last_output)
        output = self.fc(output)
        
        return output, attention_weights

# Example usage
rnn_attn_model = RNNWithAttention(
    vocab_size=10000,
    embedding_dim=128,
    hidden_size=64,
    output_size=5,
    num_layers=2
)

sample_input = torch.randint(0, 10000, (32, 50))
output, attn_weights = rnn_attn_model(sample_input)
print(f"RNN with attention output shape: {output.shape}")
print(f"Attention weights shape: {attn_weights.shape}")
```

## 9. How do you implement sequence-to-sequence models with RNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class Encoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers, dropout=0.1):
        super(Encoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, 
                           batch_first=True, dropout=dropout)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, src):
        embedded = self.dropout(self.embedding(src))
        outputs, (hidden, cell) = self.lstm(embedded)
        return outputs, (hidden, cell)

class Decoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers, dropout=0.1):
        super(Decoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, 
                           batch_first=True, dropout=dropout)
        self.fc_out = nn.Linear(hidden_size, vocab_size)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, trg, encoder_hidden, encoder_cell):
        trg = trg.unsqueeze(1)  # Add time dimension
        embedded = self.dropout(self.embedding(trg))
        
        outputs, (hidden, cell) = self.lstm(embedded, (encoder_hidden, encoder_cell))
        predictions = self.fc_out(outputs.squeeze(1))
        
        return predictions, hidden, cell

class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder):
        super(Seq2Seq, self).__init__()
        self.encoder = encoder
        self.decoder = decoder
    
    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        batch_size = src.shape[0]
        trg_len = trg.shape[1]
        trg_vocab_size = self.decoder.fc_out.out_features
        
        # Tensor to store decoder outputs
        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(src.device)
        
        # Encoder forward pass
        encoder_outputs, (hidden, cell) = self.encoder(src)
        
        # First input to the decoder is the <sos> token
        input = trg[:, 0]  # First token of target sequence
        
        for t in range(1, trg_len):
            # Use previous hidden and cell states
            output, hidden, cell = self.decoder(input, hidden, cell)
            outputs[:, t] = output
            
            # Decide whether to use teacher forcing
            teacher_force = torch.rand(1).item() < teacher_forcing_ratio
            
            # Get the highest predicted token from our predictions
            top1 = output.argmax(1)
            
            # Use either predicted token or actual token
            input = trg[:, t] if teacher_force else top1
        
        return outputs

# Enhanced Seq2Seq with Attention
class AttentionDecoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers, dropout=0.1):
        super(AttentionDecoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim + hidden_size, hidden_size, num_layers, 
                           batch_first=True, dropout=dropout)
        self.attention = nn.Linear(hidden_size * 2, 1)
        self.fc_out = nn.Linear(hidden_size * 2, vocab_size)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, trg, encoder_outputs, hidden, cell):
        trg = trg.unsqueeze(1)  # Add time dimension
        embedded = self.dropout(self.embedding(trg))
        
        # Repeat hidden state for each encoder output
        hidden_repeated = hidden[-1].unsqueeze(1).repeat(1, encoder_outputs.shape[1], 1)
        
        # Calculate attention
        energy = torch.tanh(
            self.attention(torch.cat((hidden_repeated, encoder_outputs), dim=2))
        )
        attention = F.softmax(energy.squeeze(2), dim=1)
        
        # Apply attention to encoder outputs
        weighted = torch.bmm(attention.unsqueeze(1), encoder_outputs)
        
        # Concatenate embedded input and weighted context
        rnn_input = torch.cat((embedded, weighted), dim=2)
        
        # LSTM forward pass
        output, (hidden, cell) = self.lstm(rnn_input, (hidden, cell))
        
        # Prepare output
        embedded = embedded.squeeze(1)
        output = output.squeeze(1)
        weighted = weighted.squeeze(1)
        
        prediction = self.fc_out(torch.cat((output, weighted), dim=1))
        
        return prediction, hidden, cell

class Seq2SeqWithAttention(nn.Module):
    def __init__(self, encoder, decoder):
        super(Seq2SeqWithAttention, self).__init__()
        self.encoder = encoder
        self.decoder = decoder
    
    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        batch_size = src.shape[0]
        trg_len = trg.shape[1]
        trg_vocab_size = self.decoder.fc_out.out_features
        
        # Tensor to store decoder outputs
        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(src.device)
        
        # Encoder forward pass
        encoder_outputs, (hidden, cell) = self.encoder(src)
        
        # First input to the decoder is the <sos> token
        input = trg[:, 0]
        
        for t in range(1, trg_len):
            output, hidden, cell = self.decoder(input, encoder_outputs, hidden, cell)
            outputs[:, t] = output
            
            # Decide whether to use teacher forcing
            teacher_force = torch.rand(1).item() < teacher_forcing_ratio
            top1 = output.argmax(1)
            input = trg[:, t] if teacher_force else top1
        
        return outputs

# Example usage
def create_seq2seq_model():
    # Define hyperparameters
    vocab_size = 10000
    embedding_dim = 256
    hidden_size = 512
    num_layers = 2
    
    # Create encoder and decoder
    encoder = Encoder(vocab_size, embedding_dim, hidden_size, num_layers)
    decoder = AttentionDecoder(vocab_size, embedding_dim, hidden_size, num_layers)
    
    # Create seq2seq model
    model = Seq2SeqWithAttention(encoder, decoder)
    
    return model

# Test the model
seq2seq_model = create_seq2seq_model()
src_seq = torch.randint(0, 10000, (16, 20))  # (batch, src_len)
trg_seq = torch.randint(0, 10000, (16, 25))  # (batch, trg_len)

output = seq2seq_model(src_seq, trg_seq)
print(f"Seq2Seq output shape: {output.shape}")
```

## 10. How do you implement RNNs for time series prediction?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

class TimeSeriesRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1):
        super(TimeSeriesRNN, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # LSTM layer
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        
        # Output layer
        self.fc = nn.Linear(hidden_size, output_size)
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Initialize hidden state
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # LSTM forward pass
        lstm_out, _ = self.lstm(x, (h0, c0))
        
        # Apply linear layer to all time steps
        output = self.fc(lstm_out)
        
        return output

class TimeSeriesLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, sequence_length, num_layers=1):
        super(TimeSeriesLSTM, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.sequence_length = sequence_length
        
        # LSTM layer
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        
        # Fully connected layers
        self.fc1 = nn.Linear(hidden_size, hidden_size // 2)
        self.fc2 = nn.Linear(hidden_size // 2, output_size)
        self.dropout = nn.Dropout(0.2)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Initialize hidden state
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # LSTM forward pass
        lstm_out, (hn, cn) = self.lstm(x, (h0, c0))
        
        # Use the last time step
        last_output = lstm_out[:, -1, :]
        
        # Fully connected layers
        out = self.dropout(last_output)
        out = self.fc1(out)
        out = self.relu(out)
        out = self.dropout(out)
        out = self.fc2(out)
        
        return out

class MultiStepTimeSeriesLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, prediction_steps, num_layers=1):
        super(MultiStepTimeSeriesLSTM, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.prediction_steps = prediction_steps
        
        # LSTM layer
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        
        # Output layer for multiple time steps
        self.fc = nn.Linear(hidden_size, output_size * prediction_steps)
        self.dropout = nn.Dropout(0.2)
        
    def forward(self, x):
        batch_size = x.size(0)
        
        # Initialize hidden state
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        
        # LSTM forward pass
        lstm_out, _ = self.lstm(x, (h0, c0))
        
        # Use the last time step
        last_output = lstm_out[:, -1, :]
        
        # Apply dropout
        last_output = self.dropout(last_output)
        
        # Linear layer to predict multiple time steps
        output = self.fc(last_output)
        
        # Reshape to (batch, prediction_steps, output_size)
        output = output.view(batch_size, self.prediction_steps, -1)
        
        return output

# Data preparation utilities for time series
def create_time_series_dataset(data, sequence_length, prediction_steps=1):
    """
    Create dataset for time series prediction
    """
    X, y = [], []
    for i in range(len(data) - sequence_length - prediction_steps + 1):
        X.append(data[i:(i + sequence_length)])
        y.append(data[(i + sequence_length):(i + sequence_length + prediction_steps)])
    return torch.tensor(X, dtype=torch.float32), torch.tensor(y, dtype=torch.float32)

# Example usage
def generate_sample_time_series():
    """Generate sample time series data"""
    # Create a synthetic time series with trend and seasonality
    t = np.arange(0, 1000, 0.1)
    trend = 0.001 * t
    seasonal = np.sin(0.1 * t) + 0.5 * np.sin(0.05 * t)
    noise = 0.1 * np.random.randn(len(t))
    series = trend + seasonal + noise
    return series

# Create sample time series data
sample_data = generate_sample_time_series()

# Prepare training data
sequence_length = 50
prediction_steps = 10
X, y = create_time_series_dataset(sample_data, sequence_length, prediction_steps)

print(f"Training data shape: X={X.shape}, y={y.shape}")

# Create and train the model
model = MultiStepTimeSeriesLSTM(
    input_size=1, 
    hidden_size=64, 
    output_size=1, 
    prediction_steps=prediction_steps
)

# Sample forward pass
sample_input = X[:32].unsqueeze(-1)  # Add feature dimension
sample_output = model(sample_input)
print(f"Model output shape: {sample_output.shape}")

# Loss function and optimizer
criterion = nn.MSELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

print("Time series models ready for training")
```

## 11. How do you implement RNNs for text generation?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import Dataset, DataLoader
import string
import random

class TextDataset(Dataset):
    def __init__(self, text, sequence_length=50):
        self.text = text
        self.sequence_length = sequence_length
        
        # Create character-to-index and index-to-character mappings
        self.chars = sorted(list(set(text)))
        self.char_to_idx = {ch: i for i, ch in enumerate(self.chars)}
        self.idx_to_char = {i: ch for i, ch in enumerate(self.chars)}
        self.vocab_size = len(self.chars)
        
        # Convert text to indices
        self.data = [self.char_to_idx[ch] for ch in text]
    
    def __len__(self):
        return len(self.data) - self.sequence_length
    
    def __getitem__(self, idx):
        # Input sequence
        input_seq = self.data[idx:idx + self.sequence_length]
        # Target sequence (shifted by 1)
        target_seq = self.data[idx + 1:idx + self.sequence_length + 1]
        
        return torch.tensor(input_seq), torch.tensor(target_seq)

class TextGenerationRNN(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers, dropout=0.3):
        super(TextGenerationRNN, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # Embedding layer
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # LSTM layer
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, 
                           batch_first=True, dropout=dropout)
        
        # Output layer
        self.fc = nn.Linear(hidden_size, vocab_size)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, hidden=None):
        batch_size = x.size(0)
        
        # Embedding
        embedded = self.embedding(x)
        
        # LSTM forward pass
        lstm_out, hidden = self.lstm(embedded, hidden)
        
        # Apply dropout
        lstm_out = self.dropout(lstm_out)
        
        # Output layer
        output = self.fc(lstm_out)
        
        return output, hidden

class TextGenerator:
    def __init__(self, model, char_to_idx, idx_to_char):
        self.model = model
        self.char_to_idx = char_to_idx
        self.idx_to_char = idx_to_char
        self.device = next(model.parameters()).device
    
    def generate_text(self, start_text, length=100, temperature=1.0, top_k=None):
        """
        Generate text using the trained model
        """
        self.model.eval()
        
        # Convert start text to indices
        with torch.no_grad():
            # Initialize hidden state
            hidden = None
            
            # Process start text
            start_indices = torch.tensor(
                [self.char_to_idx.get(ch, 0) for ch in start_text], 
                dtype=torch.long
            ).unsqueeze(0).to(self.device)
            
            # Forward pass through start text to update hidden state
            output, hidden = self.model(start_indices, hidden)
            
            # Get the last character's output
            last_output = output[0, -1, :] / temperature
            
            # Apply top-k filtering if specified
            if top_k:
                top_k_values, top_k_indices = torch.topk(last_output, top_k)
                last_output = torch.full_like(last_output, float('-inf'))
                last_output[top_k_indices] = top_k_values
            
            # Sample next character
            probabilities = F.softmax(last_output, dim=0)
            next_char_idx = torch.multinomial(probabilities, 1).item()
            
            generated_text = start_text + self.idx_to_char[next_char_idx]
            
            # Generate remaining characters
            for _ in range(length - 1):
                # Prepare input for next character
                next_input = torch.tensor([[next_char_idx]], dtype=torch.long).to(self.device)
                
                # Forward pass
                output, hidden = self.model(next_input, hidden)
                
                # Get output for next character
                next_output = output[0, 0, :] / temperature
                
                # Apply top-k filtering if specified
                if top_k:
                    top_k_values, top_k_indices = torch.topk(next_output, top_k)
                    next_output = torch.full_like(next_output, float('-inf'))
                    next_output[top_k_indices] = top_k_values
                
                # Sample next character
                probabilities = F.softmax(next_output, dim=0)
                next_char_idx = torch.multinomial(probabilities, 1).item()
                
                generated_text += self.idx_to_char[next_char_idx]
        
        return generated_text

# Example with a simple text generation model
def create_sample_text_generator():
    # Sample text
    sample_text = "The quick brown fox jumps over the lazy dog. " * 100
    
    # Create dataset
    dataset = TextDataset(sample_text, sequence_length=50)
    
    # Create model
    model = TextGenerationRNN(
        vocab_size=dataset.vocab_size,
        embedding_dim=128,
        hidden_size=256,
        num_layers=2
    )
    
    # Create text generator
    generator = TextGenerator(model, dataset.char_to_idx, dataset.idx_to_char)
    
    # Generate sample text
    generated = generator.generate_text("The", length=200, temperature=0.8, top_k=10)
    print(f"Generated text: {generated}")
    
    return model, dataset

# Character-level RNN for text generation
class CharLSTM(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers):
        super(CharLSTM, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, vocab_size)
    
    def forward(self, x, hidden=None):
        batch_size = x.size(0)
        
        embedded = self.embedding(x)
        lstm_out, hidden = self.lstm(embedded, hidden)
        
        # Reshape output for fully connected layer
        output = self.fc(lstm_out.contiguous().view(-1, self.hidden_size))
        output = output.view(batch_size, -1, self.fc.out_features)
        
        return output, hidden

# Usage example
print("Text generation models prepared")