# Attention Mechanisms in PyTorch - 30+ Questions and Answers

## 1. What is attention in neural networks and why is it important?

**Answer:** Attention is a mechanism that allows neural networks to focus on relevant parts of the input when making predictions. Instead of processing all input elements equally, attention computes weights that indicate the importance of each element for a specific task. This is crucial because:

1. **Selective Focus:** Networks can focus on the most relevant information
2. **Long-range Dependencies:** Attention helps capture relationships between distant elements
3. **Interpretability:** Attention weights provide insights into model decision-making
4. **Parallelization:** Unlike RNNs, attention mechanisms can be parallelized
5. **Context Awareness:** Models can consider the full context when making decisions

## 2. How do you implement basic attention in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BasicAttention(nn.Module):
    def __init__(self, hidden_size):
        super(BasicAttention, self).__init__()
        self.hidden_size = hidden_size
        self.W = nn.Linear(hidden_size, hidden_size)
        self.v = nn.Parameter(torch.rand(hidden_size))
        
    def forward(self, query, keys):
        """
        query: (batch_size, hidden_size)
        keys:  (batch_size, seq_len, hidden_size)
        """
        batch_size, seq_len = keys.size(0), keys.size(1)
        
        # Transform query to match keys dimensions
        query_transformed = self.W(query).unsqueeze(1)  # (batch, 1, hidden_size)
        
        # Calculate attention scores
        scores = torch.tanh(keys + query_transformed)  # Broadcasting: (batch, seq_len, hidden_size)
        scores = torch.sum(self.v * scores, dim=2)     # (batch, seq_len)
        
        # Calculate attention weights using softmax
        attention_weights = F.softmax(scores, dim=1)   # (batch, seq_len)
        
        # Apply attention weights to keys
        context = torch.bmm(
            attention_weights.unsqueeze(1),            # (batch, 1, seq_len)
            keys                                        # (batch, seq_len, hidden_size)
        )                                              # (batch, 1, hidden_size)
        
        return context.squeeze(1), attention_weights  # (batch, hidden_size), (batch, seq_len)

# Example usage
attention = BasicAttention(hidden_size=128)
query = torch.randn(32, 128)  # (batch_size, hidden_size)
keys = torch.randn(32, 20, 128)  # (batch_size, seq_len, hidden_size)

context, weights = attention(query, keys)
print(f"Context shape: {context.shape}")
print(f"Weights shape: {weights.shape}")
```

## 3. How do you implement scaled dot-product attention?

**Answer:**
```python
class ScaledDotProductAttention(nn.Module):
    def __init__(self, d_k):
        super(ScaledDotProductAttention, self).__init__()
        self.d_k = d_k
        
    def forward(self, query, key, value, mask=None):
        """
        query: (batch_size, num_heads, seq_len, d_k)
        key:   (batch_size, num_heads, seq_len, d_k)
        value: (batch_size, num_heads, seq_len, d_v)
        mask:  (batch_size, seq_len) or (batch_size, seq_len, seq_len)
        """
        # Calculate attention scores
        scores = torch.matmul(query, key.transpose(-2, -1))  # (batch, heads, seq_len, seq_len)
        
        # Scale scores
        scores = scores / torch.sqrt(torch.tensor(self.d_k, dtype=torch.float))
        
        # Apply mask if provided
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention weights to values
        output = torch.matmul(attention_weights, value)  # (batch, heads, seq_len, d_v)
        
        return output, attention_weights

# Multi-head attention implementation
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(MultiHeadAttention, self).__init__()
        assert d_model % num_heads == 0
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.attention = ScaledDotProductAttention(self.d_k)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query)  # (batch, seq_len, d_model)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Reshape and transpose for multi-head attention
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)  # (batch, heads, seq_len, d_k)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Apply attention
        scores, attention_weights = self.attention(Q, K, V, mask)
        
        # Reshape back to original dimensions
        scores = scores.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        
        # Final linear transformation
        output = self.W_o(scores)
        
        return self.dropout(output), attention_weights

# Example usage
multihead_attn = MultiHeadAttention(d_model=512, num_heads=8)
query = torch.randn(32, 10, 512)  # (batch, seq_len, d_model)
key = torch.randn(32, 10, 512)
value = torch.randn(32, 10, 512)

output, attention = multihead_attn(query, key, value)
print(f"Multi-head attention output shape: {output.shape}")
print(f"Attention weights shape: {attention.shape}")
```

## 4. How do you implement self-attention for sequence modeling?

**Answer:**
```python
class SelfAttention(nn.Module):
    def __init__(self, embed_size, heads):
        super(SelfAttention, self).__init__()
        self.embed_size = embed_size
        self.heads = heads
        self.head_dim = embed_size // heads
        
        assert (self.head_dim * heads == embed_size), "Embedding size needs to be divisible by heads"
        
        self.values = nn.Linear(self.head_dim, self.head_dim, bias=False)
        self.keys = nn.Linear(self.head_dim, self.head_dim, bias=False)
        self.queries = nn.Linear(self.head_dim, self.head_dim, bias=False)
        self.fc_out = nn.Linear(heads * self.head_dim, embed_size)
        
    def forward(self, values, keys, query, mask):
        # Get batch size
        batch_size = query.shape[0]
        
        # Reshape inputs
        values = values.reshape(batch_size, -1, self.heads, self.head_dim)
        keys = keys.reshape(batch_size, -1, self.heads, self.head_dim)
        query = query.reshape(batch_size, -1, self.heads, self.head_dim)
        
        values = self.values(values)
        keys = self.keys(keys)
        query = self.queries(query)
        
        # Scaled dot-product attention
        energy = torch.einsum("nqhd,nkhd->nhqk", [query, keys])
        
        if mask is not None:
            energy = energy.masked_fill(mask == 0, float("-1e20"))
        
        attention = torch.softmax(energy / (self.embed_size ** (1/2)), dim=3)
        
        out = torch.einsum("nhql,nlhd->nqhd", [attention, values]).reshape(
            batch_size, -1, self.heads * self.head_dim
        )
        
        out = self.fc_out(out)
        return out, attention

class TransformerBlock(nn.Module):
    def __init__(self, embed_size, heads, dropout, forward_expansion):
        super(TransformerBlock, self).__init__()
        self.attention = SelfAttention(embed_size, heads)
        self.norm1 = nn.LayerNorm(embed_size)
        self.norm2 = nn.LayerNorm(embed_size)
        
        self.feed_forward = nn.Sequential(
            nn.Linear(embed_size, forward_expansion * embed_size),
            nn.ReLU(),
            nn.Linear(forward_expansion * embed_size, embed_size)
        )
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, value, key, query, mask):
        attention, attention_weights = self.attention(value, key, query, mask)
        
        # Add & Norm
        x = self.dropout(self.norm1(attention + query))
        forward = self.feed_forward(x)
        out = self.dropout(self.norm2(forward + x))
        
        return out, attention_weights

class SelfAttentionSequence(nn.Module):
    def __init__(self, vocab_size, embed_size, num_layers, heads, device, forward_expansion=4, dropout=0, max_length=100):
        super(SelfAttentionSequence, self).__init__()
        self.embed_size = embed_size
        self.device = device
        
        self.word_embedding = nn.Embedding(vocab_size, embed_size)
        self.position_embedding = nn.Embedding(max_length, embed_size)
        
        self.layers = nn.ModuleList([
            TransformerBlock(
                embed_size,
                heads,
                dropout=dropout,
                forward_expansion=forward_expansion
            )
            for _ in range(num_layers)
        ])
        
        self.dropout = nn.Dropout(dropout)
        self.fc_out = nn.Linear(embed_size, vocab_size)
        
    def forward(self, x, mask):
        N, seq_length = x.shape
        positions = torch.arange(0, seq_length).expand(N, seq_length).to(self.device)
        
        out = self.dropout(self.word_embedding(x) + self.position_embedding(positions))
        
        for layer in self.layers:
            out, attention_weights = layer(out, out, out, mask)
        
        out = self.fc_out(out)
        return out, attention_weights

# Example usage
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = SelfAttentionSequence(
    vocab_size=1000,
    embed_size=256,
    num_layers=6,
    heads=8,
    device=device
)

sample_input = torch.randint(0, 1000, (32, 50))  # (batch_size, seq_len)
mask = torch.ones(32, 50)  # (batch_size, seq_len)

output, attention = model(sample_input, mask)
print(f"Self-attention output shape: {output.shape}")
print(f"Attention weights shape: {attention.shape}")
```

## 5. How do you implement Bahdanau attention (additive attention)?

**Answer:**
```python
class BahdanauAttention(nn.Module):
    def __init__(self, hidden_size):
        super(BahdanauAttention, self).__init__()
        self.hidden_size = hidden_size
        
        # Linear layers for query and key
        self.W_query = nn.Linear(hidden_size, hidden_size)
        self.W_key = nn.Linear(hidden_size, hidden_size)
        self.V = nn.Linear(hidden_size, 1)
        
    def forward(self, query, keys, mask=None):
        """
        query: (batch_size, hidden_size)
        keys:  (batch_size, seq_len, hidden_size)
        mask:  (batch_size, seq_len) - optional
        """
        batch_size, seq_len = keys.size(0), keys.size(1)
        
        # Transform query and keys
        query_transformed = self.W_query(query).unsqueeze(1)  # (batch, 1, hidden_size)
        keys_transformed = self.W_key(keys)                  # (batch, seq_len, hidden_size)
        
        # Calculate attention energies
        energies = torch.tanh(query_transformed + keys_transformed)  # (batch, seq_len, hidden_size)
        energies = self.V(energies).squeeze(2)                    # (batch, seq_len)
        
        # Apply mask if provided
        if mask is not None:
            energies = energies.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(energies, dim=1)  # (batch, seq_len)
        
        # Calculate context vector
        context = torch.bmm(
            attention_weights.unsqueeze(1),  # (batch, 1, seq_len)
            keys                              # (batch, seq_len, hidden_size)
        ).squeeze(1)                        # (batch, hidden_size)
        
        return context, attention_weights

class BahdanauAttentionDecoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers):
        super(BahdanauAttentionDecoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.vocab_size = vocab_size
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # Attention mechanism
        self.attention = BahdanauAttention(hidden_size)
        
        # LSTM with attention
        self.lstm = nn.LSTM(embedding_dim + hidden_size, hidden_size, num_layers, batch_first=True)
        
        # Output projection
        self.out = nn.Linear(hidden_size, vocab_size)
        
    def forward(self, x, encoder_outputs, hidden, cell):
        """
        x: (batch_size, 1) - single target token
        encoder_outputs: (batch_size, seq_len, hidden_size) - encoder states
        hidden, cell: LSTM states
        """
        embedded = self.embedding(x)
        
        # Calculate context using attention
        context, attention_weights = self.attention(hidden[-1], encoder_outputs)
        
        # Repeat context for each input in sequence
        context = context.unsqueeze(1).repeat(1, embedded.size(1), 1)
        
        # Concatenate embedded input and context
        lstm_input = torch.cat((embedded, context), dim=2)
        
        # LSTM forward pass
        output, (hidden, cell) = self.lstm(lstm_input, (hidden, cell))
        
        # Project to vocabulary size
        output = self.out(output)
        
        return output, hidden, cell, attention_weights

# Example usage
attention_mechanism = BahdanauAttention(hidden_size=256)
query = torch.randn(32, 256)  # (batch, hidden_size)
keys = torch.randn(32, 20, 256)  # (batch, seq_len, hidden_size)

context, weights = attention_mechanism(query, keys)
print(f"Bahdanau attention context shape: {context.shape}")
print(f"Bahdanau attention weights shape: {weights.shape}")
```

## 6. How do you implement Luong attention (multiplicative attention)?

**Answer:**
```python
class LuongAttention(nn.Module):
    def __init__(self, hidden_size, method="dot"):
        super(LuongAttention, self).__init__()
        self.hidden_size = hidden_size
        self.method = method
        
        if method == "general":
            self.W = nn.Linear(hidden_size, hidden_size, bias=False)
        elif method == "concat":
            self.W = nn.Linear(hidden_size * 2, hidden_size, bias=False)
            self.v = nn.Parameter(torch.rand(hidden_size))
    
    def forward(self, decoder_hidden, encoder_outputs, mask=None):
        """
        decoder_hidden: (batch_size, hidden_size)
        encoder_outputs: (batch_size, seq_len, hidden_size)
        mask: (batch_size, seq_len) - optional
        """
        if self.method == "general":
            decoder_hidden = self.W(decoder_hidden).unsqueeze(2)  # (batch, hidden_size, 1)
            attention_energies = torch.bmm(encoder_outputs, decoder_hidden).squeeze(2)  # (batch, seq_len)
        elif self.method == "dot":
            decoder_hidden = decoder_hidden.unsqueeze(2)  # (batch, hidden_size, 1)
            attention_energies = torch.bmm(encoder_outputs, decoder_hidden).squeeze(2)  # (batch, seq_len)
        elif self.method == "concat":
            decoder_hidden = decoder_hidden.unsqueeze(1).repeat(1, encoder_outputs.size(1), 1)  # (batch, seq_len, hidden_size)
            concat_input = torch.cat((encoder_outputs, decoder_hidden), dim=2)  # (batch, seq_len, hidden_size*2)
            attention_energies = self.v * torch.tanh(self.W(concat_input))  # (batch, seq_len, hidden_size)
            attention_energies = torch.sum(attention_energies, dim=2)  # (batch, seq_len)
        
        # Apply mask if provided
        if mask is not None:
            attention_energies = attention_energies.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(attention_energies, dim=1)  # (batch, seq_len)
        
        # Calculate context vector
        context = torch.bmm(attention_weights.unsqueeze(1), encoder_outputs).squeeze(1)  # (batch, hidden_size)
        
        return context, attention_weights

class LuongAttentionDecoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers, attention_method="dot"):
        super(LuongAttentionDecoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.vocab_size = vocab_size
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # Attention mechanism
        self.attention = LuongAttention(hidden_size, method=attention_method)
        
        # LSTM
        self.lstm = nn.LSTM(embedding_dim + hidden_size, hidden_size, num_layers, batch_first=True)
        
        # Output projection
        self.out = nn.Linear(hidden_size, vocab_size)
        
    def forward(self, x, encoder_outputs, hidden, cell):
        """
        x: (batch_size, 1) - single target token
        encoder_outputs: (batch_size, seq_len, hidden_size) - encoder states
        hidden, cell: LSTM states
        """
        embedded = self.embedding(x)
        
        # Calculate context using attention
        context, attention_weights = self.attention(hidden[-1], encoder_outputs)
        
        # Repeat context for each input in sequence
        context = context.unsqueeze(1).repeat(1, embedded.size(1), 1)
        
        # Concatenate embedded input and context
        lstm_input = torch.cat((embedded, context), dim=2)
        
        # LSTM forward pass
        output, (hidden, cell) = self.lstm(lstm_input, (hidden, cell))
        
        # Project to vocabulary size
        output = self.out(output)
        
        return output, hidden, cell, attention_weights

# Example usage
# Test different Luong attention methods
methods = ["dot", "general", "concat"]
for method in methods:
    attention = LuongAttention(hidden_size=256, method=method)
    decoder_hidden = torch.randn(32, 256)  # (batch, hidden_size)
    encoder_outputs = torch.randn(32, 20, 256)  # (batch, seq_len, hidden_size)
    
    context, weights = attention(decoder_hidden, encoder_outputs)
    print(f"Luong {method} attention - Context: {context.shape}, Weights: {weights.shape}")
```

## 7. How do you implement attention for image captioning?

**Answer:**
```python
class ImageCaptioningAttention(nn.Module):
    def __init__(self, encoder_dim, decoder_dim, attention_dim):
        super(ImageCaptioningAttention, self).__init__()
        
        self.encoder_att = nn.Linear(encoder_dim, attention_dim)  # Linear layer for encoded images
        self.decoder_att = nn.Linear(decoder_dim, attention_dim)  # Linear layer for decoder
        self.full_att = nn.Linear(attention_dim, 1)  # Linear layer to calculate values to be 'softmax-ed'
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, encoder_out, decoder_hidden):
        """
        encoder_out: (batch_size, num_pixels, encoder_dim)
        decoder_hidden: (batch_size, decoder_dim)
        """
        # Transform encoder output
        att1 = self.encoder_att(encoder_out)  # (batch_size, num_pixels, attention_dim)
        
        # Transform decoder hidden state
        att2 = self.decoder_att(decoder_hidden)  # (batch_size, attention_dim)
        att2 = att2.unsqueeze(1).repeat(1, encoder_out.size(1), 1)  # (batch_size, num_pixels, attention_dim)
        
        # Calculate attention energies
        att = self.full_att(self.relu(att1 + att2))  # (batch_size, num_pixels, 1)
        att = att.squeeze(2)  # (batch_size, num_pixels)
        
        # Calculate attention weights
        alpha = F.softmax(att, dim=1)  # (batch_size, num_pixels)
        
        # Calculate context vector
        attention_weighted_encoding = (encoder_out * alpha.unsqueeze(2)).sum(dim=1)  # (batch_size, encoder_dim)
        
        return attention_weighted_encoding, alpha

class ImageCaptioningModel(nn.Module):
    def __init__(self, vocab_size, embed_dim=512, decoder_dim=512, encoder_dim=2048, attention_dim=512):
        super(ImageCaptioningModel, self).__init__()
        
        self.vocab_size = vocab_size
        self.encoder_dim = encoder_dim
        self.decoder_dim = decoder_dim
        
        # Attention mechanism
        self.attention = ImageCaptioningAttention(encoder_dim, decoder_dim, attention_dim)
        
        # Embedding layer
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        
        # Decoder LSTM
        self.decode_step = nn.LSTMCell(embed_dim + encoder_dim, decoder_dim, bias=True)
        
        # Linear layers
        self.init_h = nn.Linear(encoder_dim, decoder_dim)  # Linear layer to find initial hidden state of LSTMCell
        self.init_c = nn.Linear(encoder_dim, decoder_dim)  # Linear layer to find initial cell state of LSTMCell
        self.f_beta = nn.Linear(decoder_dim, encoder_dim)  # Linear layer to create a sigmoid-activated gate
        self.sigmoid = nn.Sigmoid()
        self.fc = nn.Linear(decoder_dim, vocab_size)  # Linear layer to find scores over vocabulary
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, encoder_out, encoded_captions, caption_lengths):
        """
        encoder_out: (batch_size, num_pixels, encoder_dim)
        encoded_captions: (batch_size, max_caption_length)
        caption_lengths: (batch_size, 1)
        """
        batch_size = encoder_out.size(0)
        encoder_dim = encoder_out.size(-1)
        vocab_size = self.vocab_size
        
        # Flatten image
        encoder_out = encoder_out.view(batch_size, -1, encoder_dim)  # (batch_size, num_pixels, encoder_dim)
        num_pixels = encoder_out.size(1)
        
        # Sort input data by decreasing lengths
        caption_lengths, sort_ind = caption_lengths.squeeze(1).sort(dim=0, descending=True)
        encoder_out = encoder_out[sort_ind]
        encoded_captions = encoded_captions[sort_ind]
        
        # Embedding
        embeddings = self.embedding(encoded_captions)  # (batch_size, max_caption_length, embed_dim)
        
        # Initialize LSTM state
        mean_encoder_out = encoder_out.mean(dim=1)  # (batch_size, encoder_dim)
        h = self.init_h(mean_encoder_out)  # (batch_size, decoder_dim)
        c = self.init_c(mean_encoder_out)  # (batch_size, decoder_dim)
        
        # We won't decode at the <end> position, since we've finished generating as soon as we generate <end>
        decode_lengths = (caption_lengths - 1).tolist()
        
        # Create tensors to hold word predicion scores and alphas
        predictions = torch.zeros(batch_size, max(decode_lengths), vocab_size).to(encoder_out.device)
        alphas = torch.zeros(batch_size, max(decode_lengths), num_pixels).to(encoder_out.device)
        
        # At each time-step, decode by
        # attention-weighing the encoder's output based on the decoder's previous hidden state output
        # then generate a new word in the decoder with the previous word and the attention weighted encoding
        for t in range(max(decode_lengths)):
            batch_size_t = sum([l > t for l in decode_lengths])
            
            attention_weighted_encoding, alpha = self.attention(encoder_out[:batch_size_t], h[:batch_size_t])
            
            gate = self.sigmoid(self.f_beta(h[:batch_size_t]))  # gating scalar, (batch_size_t, encoder_dim)
            attention_weighted_encoding = gate * attention_weighted_encoding
            
            h, c = self.decode_step(
                torch.cat([embeddings[:batch_size_t, t, :], attention_weighted_encoding], dim=1),
                (h[:batch_size_t], c[:batch_size_t])
            )  # (batch_size_t, decoder_dim)
            
            preds = self.fc(self.dropout(h))  # (batch_size_t, vocab_size)
            predictions[:batch_size_t, t, :] = preds
            alphas[:batch_size_t, t, :] = alpha
        
        return predictions, encoded_captions, decode_lengths, alphas, sort_ind

# Example usage for image captioning attention
image_attention = ImageCaptioningAttention(encoder_dim=2048, decoder_dim=512, attention_dim=512)
encoder_out = torch.randn(8, 196, 2048)  # 8 images, 14x14 spatial locations, 2048 features each
decoder_hidden = torch.randn(8, 512)  # 8 images, 512-dim hidden state

weighted_encoding, attention_weights = image_attention(encoder_out, decoder_hidden)
print(f"Image attention weighted encoding: {weighted_encoding.shape}")
print(f"Attention weights: {attention_weights.shape}")
```

## 8. How do you implement location-based attention?

**Answer:**
```python
class LocationBasedAttention(nn.Module):
    def __init__(self, hidden_size, location_features=32):
        super(LocationBasedAttention, self).__init__()
        self.hidden_size = hidden_size
        self.location_features = location_features
        
        # Previous attention weights processing
        self.location_conv = nn.Conv1d(1, location_features, kernel_size=3, padding=1, bias=False)
        self.W_location = nn.Linear(location_features, hidden_size, bias=False)
        self.W_process = nn.Linear(hidden_size, hidden_size, bias=False)
        self.V = nn.Linear(hidden_size, 1, bias=False)
        
    def forward(self, decoder_hidden, encoder_outputs, prev_attention_weights):
        """
        decoder_hidden: (batch_size, hidden_size)
        encoder_outputs: (batch_size, seq_len, hidden_size)
        prev_attention_weights: (batch_size, seq_len)
        """
        batch_size, seq_len = encoder_outputs.size(0), encoder_outputs.size(1)
        
        # Process previous attention weights
        prev_attn_expanded = prev_attention_weights.unsqueeze(1)  # (batch, 1, seq_len)
        location_features = self.location_conv(prev_attn_expanded)  # (batch, location_features, seq_len)
        location_features = location_features.transpose(1, 2)  # (batch, seq_len, location_features)
        
        # Process location features
        processed_location = self.W_location(location_features)  # (batch, seq_len, hidden_size)
        
        # Process decoder hidden state
        processed_hidden = self.W_process(decoder_hidden).unsqueeze(1)  # (batch, 1, hidden_size)
        
        # Calculate attention energies
        energies = torch.tanh(processed_hidden + encoder_outputs + processed_location)
        energies = self.V(energies).squeeze(2)  # (batch, seq_len)
        
        # Calculate attention weights
        attention_weights = F.softmax(energies, dim=1)  # (batch, seq_len)
        
        # Calculate context vector
        context = torch.bmm(attention_weights.unsqueeze(1), encoder_outputs).squeeze(1)  # (batch, hidden_size)
        
        return context, attention_weights

class LocationBasedAttentionDecoder(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_size, num_layers):
        super(LocationBasedAttentionDecoder, self).__init__()
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.attention = LocationBasedAttention(hidden_size)
        self.lstm = nn.LSTM(embedding_dim + hidden_size, hidden_size, num_layers, batch_first=True)
        self.out = nn.Linear(hidden_size, vocab_size)
        
    def forward(self, x, encoder_outputs, hidden, cell, prev_attention_weights):
        embedded = self.embedding(x)
        
        # Calculate context using location-based attention
        context, attention_weights = self.attention(
            hidden[-1], encoder_outputs, prev_attention_weights
        )
        
        # Repeat context for each input in sequence
        context = context.unsqueeze(1).repeat(1, embedded.size(1), 1)
        
        # Concatenate embedded input and context
        lstm_input = torch.cat((embedded, context), dim=2)
        
        # LSTM forward pass
        output, (hidden, cell) = self.lstm(lstm_input, (hidden, cell))
        
        # Project to vocabulary size
        output = self.out(output)
        
        return output, hidden, cell, attention_weights

# Example usage
location_attention = LocationBasedAttention(hidden_size=256, location_features=32)
decoder_hidden = torch.randn(32, 256)
encoder_outputs = torch.randn(32, 20, 256)
prev_attention = torch.ones(32, 20) / 20  # Uniform distribution

context, weights = location_attention(decoder_hidden, encoder_outputs, prev_attention)
print(f"Location-based attention context: {context.shape}")
print(f"Location-based attention weights: {weights.shape}")
```

## 9. How do you implement hierarchical attention networks?

**Answer:**
```python
class HierarchicalAttention(nn.Module):
    def __init__(self, word_hidden_size, sentence_hidden_size, attention_size):
        super(HierarchicalAttention, self).__init__()
        
        # Word-level attention
        self.word_attention = nn.Linear(word_hidden_size, attention_size)
        self.word_context = nn.Linear(attention_size, 1, bias=False)
        
        # Sentence-level attention
        self.sentence_attention = nn.Linear(sentence_hidden_size, attention_size)
        self.sentence_context = nn.Linear(attention_size, 1, bias=False)
        
        # Activation and normalization
        self.tanh = nn.Tanh()
        self.softmax = nn.Softmax(dim=1)
        
    def word_attention_forward(self, word_embeddings):
        """
        word_embeddings: (batch_size, max_sent_len, word_hidden_size)
        """
        # Apply attention to word embeddings
        word_attn = self.tanh(self.word_attention(word_embeddings))  # (batch, sent_len, attention_size)
        word_attn_weights = self.softmax(self.word_context(word_attn).squeeze(2))  # (batch, sent_len)
        
        # Calculate sentence vector as weighted sum of word embeddings
        sentence_vector = torch.bmm(
            word_attn_weights.unsqueeze(1),  # (batch, 1, sent_len)
            word_embeddings  # (batch, sent_len, word_hidden_size)
        ).squeeze(1)  # (batch, word_hidden_size)
        
        return sentence_vector, word_attn_weights
    
    def sentence_attention_forward(self, sentence_vectors):
        """
        sentence_vectors: (batch_size, max_doc_len, sentence_hidden_size)
        """
        # Apply attention to sentence vectors
        sent_attn = self.tanh(self.sentence_attention(sentence_vectors))  # (batch, doc_len, attention_size)
        sent_attn_weights = self.softmax(self.sentence_context(sent_attn).squeeze(2))  # (batch, doc_len)
        
        # Calculate document vector as weighted sum of sentence vectors
        doc_vector = torch.bmm(
            sent_attn_weights.unsqueeze(1),  # (batch, 1, doc_len)
            sentence_vectors  # (batch, doc_len, sentence_hidden_size)
        ).squeeze(1)  # (batch, sentence_hidden_size)
        
        return doc_vector, sent_attn_weights

class HierarchicalAttentionNetwork(nn.Module):
    def __init__(self, vocab_size, embedding_dim, word_hidden_size, sentence_hidden_size, attention_size, num_classes):
        super(HierarchicalAttentionNetwork, self).__init__()
        
        # Embedding layer
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        
        # Word-level BiLSTM
        self.word_bilstm = nn.LSTM(
            embedding_dim, 
            word_hidden_size, 
            batch_first=True, 
            bidirectional=True
        )
        
        # Sentence-level BiLSTM
        self.sentence_bilstm = nn.LSTM(
            word_hidden_size * 2,  # *2 for bidirectional
            sentence_hidden_size, 
            batch_first=True, 
            bidirectional=True
        )
        
        # Hierarchical attention
        self.hier_attention = HierarchicalAttention(
            word_hidden_size * 2,  # *2 for bidirectional
            sentence_hidden_size * 2,  # *2 for bidirectional
            attention_size
        )
        
        # Classification layer
        self.classifier = nn.Linear(sentence_hidden_size * 2, num_classes)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, document_batch):
        """
        document_batch: (batch_size, num_sentences, max_sent_len)
        """
        batch_size, num_sentences, max_sent_len = document_batch.size()
        
        # Reshape to process all sentences together
        sentences = document_batch.view(-1, max_sent_len)  # (batch_size * num_sentences, max_sent_len)
        
        # Embed words
        embedded_words = self.embedding(sentences)  # (batch_size * num_sentences, max_sent_len, embedding_dim)
        
        # Word-level BiLSTM
        word_outputs, _ = self.word_bilstm(embedded_words)  # (batch_size * num_sentences, max_sent_len, word_hidden_size*2)
        
        # Apply word-level attention to get sentence vectors
        sentence_vectors = []
        for i in range(batch_size * num_sentences):
            sent_vec, _ = self.hier_attention.word_attention_forward(
                word_outputs[i].unsqueeze(0)
            )
            sentence_vectors.append(sent_vec)
        
        # Reshape back to document structure
        sentence_vectors = torch.stack(sentence_vectors, dim=0)  # (batch_size * num_sentences, word_hidden_size*2)
        sentence_vectors = sentence_vectors.view(batch_size, num_sentences, -1)  # (batch_size, num_sentences, word_hidden_size*2)
        
        # Sentence-level BiLSTM
        sent_outputs, _ = self.sentence_bilstm(sentence_vectors)  # (batch_size, num_sentences, sentence_hidden_size*2)
        
        # Apply sentence-level attention to get document vector
        doc_vector, sent_attention_weights = self.hier_attention.sentence_attention_forward(sent_outputs)
        
        # Final classification
        output = self.classifier(self.dropout(doc_vector))
        
        return output, sent_attention_weights

# Example usage
hier_attn_model = HierarchicalAttentionNetwork(
    vocab_size=10000,
    embedding_dim=100,
    word_hidden_size=128,
    sentence_hidden_size=128,
    attention_size=50,
    num_classes=5
)

# Sample document batch: (batch_size=2, num_sentences=10, max_sent_len=20)
sample_docs = torch.randint(0, 10000, (2, 10, 20))
output, attention_weights = hier_attn_model(sample_docs)
print(f"Hierarchical attention output: {output.shape}")
print(f"Attention weights: {attention_weights.shape}")
```

## 10. How do you implement multi-scale attention?

**Answer:**
```python
class MultiScaleAttention(nn.Module):
    def __init__(self, d_model, scales=[1, 3, 5, 7], num_heads=8):
        super(MultiScaleAttention, self).__init__()
        self.d_model = d_model
        self.scales = scales
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        assert d_model % num_heads == 0
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Convolution layers for multi-scale processing
        self.scale_convs = nn.ModuleList([
            nn.Conv1d(d_model, d_model, kernel_size=scale, padding=scale//2, groups=num_heads)
            for scale in scales
        ])
        
        # Attention weight calculation
        self.attention_weights = nn.Linear(d_model, num_heads)
        
        self.dropout = nn.Dropout(0.1)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Reshape for multi-head attention
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / torch.sqrt(torch.tensor(self.d_k, dtype=torch.float))
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention weights
        output = torch.matmul(attention_weights, V)
        
        # Reshape back
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        
        # Multi-scale processing
        multi_scale_outputs = []
        output_permuted = output.transpose(1, 2)  # (batch, d_model, seq_len)
        
        for conv in self.scale_convs:
            scale_output = F.relu(conv(output_permuted))
            multi_scale_outputs.append(scale_output.transpose(1, 2))  # Back to (batch, seq_len, d_model)
        
        # Combine multi-scale outputs
        combined_output = torch.stack(multi_scale_outputs, dim=0).mean(dim=0)
        
        # Final linear transformation
        final_output = self.W_o(combined_output)
        
        return self.dropout(final_output)

class MultiScaleTransformerBlock(nn.Module):
    def __init__(self, d_model, scales, num_heads, d_ff, dropout=0.1):
        super(MultiScaleTransformerBlock, self).__init__()
        
        self.attention = MultiScaleAttention(d_model, scales, num_heads)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        
        # Feed forward network
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Linear(d_ff, d_model),
            nn.Dropout(dropout)
        )
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Multi-scale attention
        attn_output = self.attention(x, x, x, mask)
        
        # Add & Norm
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed Forward
        ff_output = self.ffn(x)
        
        # Add & Norm
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

# Example usage
multi_scale_attn = MultiScaleAttention(d_model=512, scales=[1, 3, 5, 7], num_heads=8)
query = torch.randn(32, 20, 512)  # (batch, seq_len, d_model)
key = torch.randn(32, 20, 512)
value = torch.randn(32, 20, 512)

output = multi_scale_attn(query, key, value)
print(f"Multi-scale attention output: {output.shape}")

# Example transformer block with multi-scale attention
transformer_block = MultiScaleTransformerBlock(
    d_model=512,
    scales=[1, 3, 5, 7],
    num_heads=8,
    d_ff=2048
)

block_output = transformer_block(query)
print(f"Transformer block output: {block_output.shape}")
```

## 11. How do you implement sparse attention mechanisms?

**Answer:**
```python
class SparseAttention(nn.Module):
    def __init__(self, d_model, num_heads, block_size=64, local_attn_len=64, dropout=0.1):
        super(SparseAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.block_size = block_size
        self.local_attn_len = local_attn_len
        self.d_k = d_model // num_heads
        
        assert d_model % num_heads == 0
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Reshape for multi-head attention
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        seq_len = Q.size(2)
        
        # Apply sparse attention: only attend to local window
        # For each position, only attend to positions within local_attn_len
        sparse_scores = torch.zeros_like(Q.matmul(K.transpose(-2, -1)))
        
        for i in range(0, seq_len, self.block_size):
            end_idx = min(i + self.block_size, seq_len)
            
            # Get local attention for this block
            q_block = Q[:, :, i:end_idx, :]
            k_block = K[:, :, 
                       max(0, i - self.local_attn_len//2):min(seq_len, i + self.local_attn_len//2), :]
            v_block = V[:, :, 
                       max(0, i - self.local_attn_len//2):min(seq_len, i + self.local_attn_len//2), :]
            
            # Calculate attention scores for this local window
            local_scores = torch.matmul(q_block, k_block.transpose(-2, -1)) / torch.sqrt(torch.tensor(self.d_k, dtype=torch.float))
            
            # Apply mask if provided
            if mask is not None:
                local_mask = mask[:, max(0, i - self.local_attn_len//2):min(seq_len, i + self.local_attn_len//2)]
                local_scores = local_scores.masked_fill(local_mask == 0, float('-inf'))
            
            # Calculate attention weights
            local_attention = F.softmax(local_scores, dim=-1)
            local_attention = self.dropout(local_attention)
            
            # Apply attention
            local_output = torch.matmul(local_attention, v_block)
            
            # Place back into full output tensor
            sparse_scores[:, :, i:end_idx, 
                         max(0, i - self.local_attn_len//2):min(seq_len, i + self.local_attn_len//2)] = local_scores
        
        # Reshape back
        output = sparse_scores.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return self.dropout(output)

class FixedSparseAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(FixedSparseAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Reshape for multi-head attention
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        seq_len = Q.size(2)
        
        # Create sparse attention pattern (only attend to adjacent positions)
        scores = torch.zeros(batch_size, self.num_heads, seq_len, seq_len).to(Q.device)
        
        # Fill in sparse attention scores
        for i in range(seq_len):
            # Attend to current position and adjacent positions
            start_idx = max(0, i - 2)
            end_idx = min(seq_len, i + 3)  # Include current position and 2 on each side
            
            scores[:, :, i, start_idx:end_idx] = (
                Q[:, :, i:i+1, :] @ K[:, :, start_idx:end_idx, :].transpose(-2, -1)
            ) / torch.sqrt(torch.tensor(self.d_k, dtype=torch.float))
        
        # Apply mask if provided
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape back
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return self.dropout(output)

# Example usage
sparse_attn = FixedSparseAttention(d_model=512, num_heads=8)
query = torch.randn(32, 100, 512)  # (batch, seq_len, d_model)
key = torch.randn(32, 100, 512)
value = torch.randn(32, 100, 512)

output = sparse_attn(query, key, value)
print(f"Sparse attention output: {output.shape}")
```

## 12. How do you implement attention with memory networks?

**Answer:**
```python
class MemoryNetworkAttention(nn.Module):
    def __init__(self, input_size, memory_size, memory_vector_dim):
        super(MemoryNetworkAttention, self).__init__()
        
        self.input_size = input_size
        self.memory_size = memory_size
        self.memory_vector_dim = memory_vector_dim
        
        # Linear transformation for input
        self.input_linear = nn.Linear(input_size, memory_vector_dim)
        
        # Memory matrix (initialized randomly)
        self.memory = nn.Parameter(torch.randn(memory_size, memory_vector_dim))
        
        # For attention computation
        self.W = nn.Linear(memory_vector_dim, memory_vector_dim)
        self.V = nn.Linear(memory_vector_dim, 1)
        
    def forward(self, input_vec, memory_state=None):
        """
        input_vec: (batch_size, input_size)
        memory_state: (batch_size, memory_size, memory_vector_dim) - optional
        """
        batch_size = input_vec.size(0)
        
        # Transform input vector
        input_transformed = F.relu(self.input_linear(input_vec))  # (batch, memory_vector_dim)
        
        # If no external memory state provided, use internal memory
        if memory_state is None:
            memory_expanded = self.memory.unsqueeze(0).expand(batch_size, -1, -1)  # (batch, memory_size, memory_vector_dim)
        else:
            memory_expanded = memory_state
        
        # Calculate attention scores between input and memory
        input_repeated = input_transformed.unsqueeze(1).expand(-1, self.memory_size, -1)  # (batch, memory_size, memory_vector_dim)
        
        # Combine input and memory for attention calculation
        combined = torch.tanh(self.W(input_repeated + memory_expanded))  # (batch, memory_size, memory_vector_dim)
        attention_scores = self.V(combined).squeeze(2)  # (batch, memory_size)
        
        # Calculate attention weights
        attention_weights = F.softmax(attention_scores, dim=1)  # (batch, memory_size)
        
        # Calculate context vector as weighted sum of memory
        context = torch.bmm(attention_weights.unsqueeze(1), memory_expanded).squeeze(1)  # (batch, memory_vector_dim)
        
        # Update memory (simple implementation - replace least attended memory slot)
        min_attention_idx = torch.argmin(attention_weights, dim=1)  # (batch,)
        updated_memory = memory_expanded.clone()
        
        for i in range(batch_size):
            # Replace the least attended memory slot with the current input
            updated_memory[i, min_attention_idx[i]] = input_transformed[i]
        
        return context, attention_weights, updated_memory

class NeuralTuringMachine(nn.Module):
    def __init__(self, input_size, output_size, controller_size, memory_size=128, memory_vector_dim=20):
        super(NeuralTuringMachine, self).__init__()
        
        self.controller_size = controller_size
        self.memory_size = memory_size
        self.memory_vector_dim = memory_vector_dim
        
        # Controller network (e.g., LSTM)
        self.controller = nn.LSTMCell(input_size + memory_vector_dim, controller_size)
        
        # Memory attention mechanism
        self.memory_attention = MemoryNetworkAttention(controller_size, memory_size, memory_vector_dim)
        
        # Output projection
        self.output = nn.Linear(controller_size, output_size)
        
    def forward(self, input_sequence):
        """
        input_sequence: (batch_size, seq_len, input_size)
        """
        batch_size, seq_len = input_sequence.size(0), input_sequence.size(1)
        
        # Initialize controller state
        hx = torch.zeros(batch_size, self.controller_size).to(input_sequence.device)
        cx = torch.zeros(batch_size, self.controller_size).to(input_sequence.device)
        
        # Initialize memory
        memory_state = torch.randn(batch_size, self.memory_size, self.memory_vector_dim).to(input_sequence.device)
        
        outputs = []
        
        for t in range(seq_len):
            # Current input
            current_input = input_sequence[:, t, :]  # (batch, input_size)
            
            # Get memory content using attention
            memory_content, attention_weights, updated_memory = self.memory_attention(hx, memory_state)
            
            # Controller input is concatenation of input and memory content
            controller_input = torch.cat([current_input, memory_content], dim=1)  # (batch, input_size + memory_vector_dim)
            
            # Controller forward pass
            hx, cx = self.controller(controller_input, (hx, cx))
            
            # Update memory state
            memory_state = updated_memory
            
            # Generate output
            output = self.output(hx)
            outputs.append(output)
        
        # Stack outputs across time
        outputs = torch.stack(outputs, dim=1)  # (batch, seq_len, output_size)
        
        return outputs

# Example usage
memory_attn = MemoryNetworkAttention(input_size=128, memory_size=50, memory_vector_dim=64)
input_vec = torch.randn(32, 128)  # (batch, input_size)

context, weights, updated_memory = memory_attn(input_vec)
print(f"Memory attention context: {context.shape}")
print(f"Attention weights: {weights.shape}")
print(f"Updated memory: {updated_memory.shape}")

# Example Neural Turing Machine
ntm = NeuralTuringMachine(
    input_size=50,
    output_size=10,
    controller_size=128,
    memory_size=64,
    memory_vector_dim=32
)

input_seq = torch.randn(16, 10, 50)  # (batch, seq_len, input_size)
ntm_output = ntm(input_seq)
print(f"NTM output: {ntm_output.shape}")
```

## 13. How do you implement attention for question answering?

**Answer:**
```python
class BiDAFAttention(nn.Module):
    """
    Bi-Directional Attention Flow for machine comprehension
    """
    def __init__(self, hidden_size):
        super(BiDAFAttention, self).__init__()
        self.hidden_size = hidden_size
        
        # Linear transformations for similarity matrix computation
        self.W_sim = nn.Linear(3 * hidden_size, 1, bias=False)
        
    def forward(self, context, query, context_mask=None, query_mask=None):
        """
        context: (batch_size, context_len, hidden_size)
        query: (batch_size, query_len, hidden_size)
        """
        batch_size, context_len, hidden_size = context.size()
        query_len = query.size(1)
        
        # Expand tensors for similarity computation
        context_expanded = context.unsqueeze(2).expand(-1, -1, query_len, -1)  # (batch, context_len, query_len, hidden_size)
        query_expanded = query.unsqueeze(1).expand(-1, context_len, -1, -1)    # (batch, context_len, query_len, hidden_size)
        
        # Concatenate context, query, and their element-wise product
        similarity_input = torch.cat([
            context_expanded,
            query_expanded,
            context_expanded * query_expanded
        ], dim=3)  # (batch, context_len, query_len, 3*hidden_size)
        
        # Compute similarity matrix
        similarity_matrix = self.W_sim(similarity_input).squeeze(3)  # (batch, context_len, query_len)
        
        # Apply masks if provided
        if context_mask is not None:
            context_mask = context_mask.unsqueeze(2)  # (batch, context_len, 1)
            similarity_matrix = similarity_matrix.masked_fill(context_mask == 0, float('-inf'))
        
        if query_mask is not None:
            query_mask = query_mask.unsqueeze(1)  # (batch, 1, query_len)
            similarity_matrix = similarity_matrix.masked_fill(query_mask == 0, float('-inf'))
        
        # Context-to-query attention
        context_to_query_attn = F.softmax(similarity_matrix, dim=2)  # (batch, context_len, query_len)
        
        # Query-to-context attention
        query_to_context_attn = F.softmax(torch.max(similarity_matrix, dim=2)[0], dim=1)  # (batch, context_len)
        
        # Apply context-to-query attention
        c2q_output = torch.bmm(context_to_query_attn, query)  # (batch, context_len, hidden_size)
        
        # Apply query-to-context attention
        q2c_output = torch.bmm(query_to_context_attn.unsqueeze(1), context).repeat(1, context_len, 1)  # (batch, context_len, hidden_size)
        
        # Combine outputs
        output = torch.cat([context, c2q_output, context * c2q_output, context * q2c_output], dim=2)
        
        return output, context_to_query_attn, query_to_context_attn

class QuestionAnsweringAttention(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, max_context_len=400, max_query_len=50):
        super(QuestionAnsweringAttention, self).__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_size)
        
        # BiLSTM for context and query encoding
        self.context_encoder = nn.LSTM(
            embed_size, hidden_size, batch_first=True, bidirectional=True
        )
        self.query_encoder = nn.LSTM(
            embed_size, hidden_size, batch_first=True, bidirectional=True
        )
        
        # BiDAF attention mechanism
        self.attention = BiDAFAttention(hidden_size * 2)  # *2 for bidirectional
        
        # Output layers
        self.output_proj = nn.Linear(hidden_size * 8, hidden_size * 2)  # *8 because of BiDAF output
        self.start_pred = nn.Linear(hidden_size * 2, 1)
        self.end_pred = nn.Linear(hidden_size * 2, 1)
        
    def forward(self, context, query, context_mask=None, query_mask=None):
        """
        context: (batch_size, context_len)
        query: (batch_size, query_len)
        """
        # Embeddings
        context_embedded = self.embedding(context)  # (batch, context_len, embed_size)
        query_embedded = self.embedding(query)      # (batch, query_len, embed_size)
        
        # Encode with BiLSTM
        context_encoded, _ = self.context_encoder(context_embedded)  # (batch, context_len, hidden_size*2)
        query_encoded, _ = self.query_encoder(query_embedded)        # (batch, query_len, hidden_size*2)
        
        # Apply BiDAF attention
        attended_output, c2q_attn, q2c_attn = self.attention(
            context_encoded, query_encoded, context_mask, query_mask
        )  # (batch, context_len, hidden_size*8)
        
        # Project for output
        projected = F.relu(self.output_proj(attended_output))  # (batch, context_len, hidden_size*2)
        
        # Predict start and end positions
        start_logits = self.start_pred(projected).squeeze(2)  # (batch, context_len)
        end_logits = self.end_pred(projected).squeeze(2)      # (batch, context_len)
        
        # Apply masks
        if context_mask is not None:
            start_logits = start_logits.masked_fill(context_mask == 0, float('-inf'))
            end_logits = end_logits.masked_fill(context_mask == 0, float('-inf'))
        
        return start_logits, end_logits, c2q_attn, q2c_attn

# Example usage
qa_attention = QuestionAnsweringAttention(
    vocab_size=10000,
    embed_size=300,
    hidden_size=100
)

context = torch.randint(0, 10000, (8, 200))  # (batch, context_len)
query = torch.randint(0, 10000, (8, 50))     # (batch, query_len)

context_mask = torch.ones(8, 200)
query_mask = torch.ones(8, 50)

start_logits, end_logits, c2q_attn, q2c_attn = qa_attention(context, query, context_mask, query_mask)
print(f"Start logits: {start_logits.shape}")
print(f"End logits: {end_logits.shape}")
print(f"C2Q attention: {c2q_attn.shape}")
print(f"Q2C attention: {q2c_attn.shape}")
```

## 14. How do you implement attention for machine translation?

**Answer:**
```python
class TranslationAttention(nn.Module):
    def __init__(self, hidden_size, dropout=0.1):
        super(TranslationAttention, self).__init__()
        self.hidden_size = hidden_size
        self.dropout = nn.Dropout(dropout)
        
        # Attention weights computation
        self.attn_combine = nn.Linear(hidden_size * 2, hidden_size)
        self.attn_weights = nn.Linear(hidden_size * 2, hidden_size)
        self.tanh = nn.Tanh()
        self.softmax = nn.Softmax(dim=1)
        
    def forward(self, hidden, encoder_outputs, mask=None):
        """
        hidden: (batch_size, hidden_size) - current decoder hidden state
        encoder_outputs: (batch_size, max_length, hidden_size) - encoder outputs
        mask: (batch_size, max_length) - optional mask for padding
        """
        batch_size = encoder_outputs.size(0)
        max_len = encoder_outputs.size(1)
        
        # Repeat hidden state for each encoder output
        hidden_repeated = hidden.unsqueeze(1).repeat(1, max_len, 1)  # (batch, max_len, hidden_size)
        
        # Calculate attention energies
        attn_energies = self.tanh(self.attn_weights(
            torch.cat((hidden_repeated, encoder_outputs), 2)
        ))  # (batch, max_len, hidden_size)
        
        # Calculate attention weights
        attn_weights = torch.sum(attn_energies, dim=2)  # (batch, max_len)
        
        # Apply mask if provided
        if mask is not None:
            attn_weights = attn_weights.masked_fill(mask == 0, float('-inf'))
        
        attn_weights = self.softmax(attn_weights)
        attn_weights = self.dropout(attn_weights)
        
        # Apply attention weights to encoder outputs
        context = torch.bmm(attn_weights.unsqueeze(1), encoder_outputs)  # (batch, 1, hidden_size)
        context = context.squeeze(1)  # (batch, hidden_size)
        
        # Combine context and current hidden state
        combined = torch.cat((context, hidden), dim=1)  # (batch, hidden_size*2)
        combined_output = self.tanh(self.attn_combine(combined))  # (batch, hidden_size)
        
        return combined_output, attn_weights

class TranslationEncoder(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, num_layers, dropout=0.1):
        super(TranslationEncoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.gru = nn.GRU(embed_size, hidden_size, num_layers, batch_first=True, dropout=dropout)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, src, hidden=None):
        embedded = self.dropout(self.embedding(src))
        outputs, hidden = self.gru(embedded, hidden)
        return outputs, hidden

class TranslationDecoder(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, num_layers, dropout=0.1):
        super(TranslationDecoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.attention = TranslationAttention(hidden_size, dropout)
        self.gru = nn.GRU(embed_size + hidden_size, hidden_size, num_layers, batch_first=True, dropout=dropout)
        self.out = nn.Linear(hidden_size, vocab_size)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, input, hidden, encoder_outputs, mask=None):
        embedded = self.dropout(self.embedding(input))
        
        # Apply attention
        attended_output, attn_weights = self.attention(hidden[-1], encoder_outputs, mask)
        
        # Concatenate embedded input and attended output
        rnn_input = torch.cat((embedded, attended_output.unsqueeze(1)), dim=2)
        
        # GRU forward pass
        output, hidden = self.gru(rnn_input, hidden)
        
        # Output projection
        output = self.out(output.squeeze(1))
        
        return output, hidden, attn_weights

class TranslationModel(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, embed_size, hidden_size, num_layers, dropout=0.1):
        super(TranslationModel, self).__init__()
        self.encoder = TranslationEncoder(src_vocab_size, embed_size, hidden_size, num_layers, dropout)
        self.decoder = TranslationDecoder(tgt_vocab_size, embed_size, hidden_size, num_layers, dropout)
        
    def forward(self, src, tgt, teacher_forcing_ratio=0.5):
        batch_size = src.size(0)
        tgt_len = tgt.size(1)
        tgt_vocab_size = self.decoder.out.out_features
        
        # Encoder forward pass
        encoder_outputs, hidden = self.encoder(src)
        
        # Prepare output tensor
        outputs = torch.zeros(batch_size, tgt_len, tgt_vocab_size).to(src.device)
        
        # First input to decoder
        input = tgt[:, 0].unsqueeze(1)  # (batch, 1)
        
        for t in range(1, tgt_len):
            output, hidden, attn_weights = self.decoder(input, hidden, encoder_outputs)
            outputs[:, t] = output
            
            # Teacher forcing
            teacher_force = torch.rand(1).item() < teacher_forcing_ratio
            top1 = output.argmax(1)
            input = tgt[:, t].unsqueeze(1) if teacher_force else top1.unsqueeze(1)
        
        return outputs

# Transformer-based attention for translation
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000):
        super(PositionalEncoding, self).__init__()
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        
        div_term = torch.exp(torch.arange(0, d_model, 2).float() *
                            -(math.log(10000.0) / d_model))
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        self.register_buffer('pe', pe.unsqueeze(0))
        
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]

import math

class TransformerTranslation(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, d_model=512, num_heads=8, num_layers=6, d_ff=2048, dropout=0.1):
        super(TransformerTranslation, self).__init__()
        
        self.d_model = d_model
        
        # Embeddings and positional encoding
        self.src_embedding = nn.Embedding(src_vocab_size, d_model)
        self.tgt_embedding = nn.Embedding(tgt_vocab_size, d_model)
        self.pos_encoding = PositionalEncoding(d_model)
        
        # Transformer layers
        self.transformer = nn.Transformer(
            d_model=d_model,
            nhead=num_heads,
            num_encoder_layers=num_layers,
            num_decoder_layers=num_layers,
            dim_feedforward=d_ff,
            dropout=dropout
        )
        
        # Output projection
        self.generator = nn.Linear(d_model, tgt_vocab_size)
        
    def forward(self, src, tgt, src_mask=None, tgt_mask=None, memory_mask=None,
                src_key_padding_mask=None, tgt_key_padding_mask=None, memory_key_padding_mask=None):
        
        # Embed and add positional encoding
        src_embedded = self.pos_encoding(self.src_embedding(src) * math.sqrt(self.d_model))
        tgt_embedded = self.pos_encoding(self.tgt_embedding(tgt) * math.sqrt(self.d_model))
        
        # Transformer forward
        output = self.transformer(
            src_embedded.transpose(0, 1),  # (seq_len, batch_size, d_model)
            tgt_embedded.transpose(0, 1),  # (seq_len, batch_size, d_model)
            src_mask, tgt_mask, memory_mask,
            src_key_padding_mask, tgt_key_padding_mask, memory_key_padding_mask
        )
        
        # Output projection
        output = self.generator(output.transpose(0, 1))  # (batch_size, seq_len, tgt_vocab_size)
        
        return output

# Example usage
translation_model = TransformerTranslation(
    src_vocab_size=5000,
    tgt_vocab_size=5000,
    d_model=512,
    num_heads=8,
    num_layers=6
)

src = torch.randint(1, 5000, (16, 30))  # (batch, src_len)
tgt = torch.randint(1, 5000, (16, 30))  # (batch, tgt_len)

output = translation_model(src, tgt)
print(f"Translation model output: {output.shape}")
```

## 15. How do you implement attention dropout and regularization?

**Answer:**
```python
class AttentionWithDropout(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1, attention_dropout=0.1):
        super(AttentionWithDropout, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Dropout layers
        self.dropout = nn.Dropout(dropout)
        self.attention_dropout = nn.Dropout(attention_dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights with dropout
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.attention_dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return self.dropout(output), attention_weights

class AttentionRegularization(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(AttentionRegularization, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply entropy regularization
        entropy_reg = -torch.sum(attention_weights * torch.log(attention_weights + 1e-8), dim=-1).mean()
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        output = self.dropout(output)
        
        return output, attention_weights, entropy_reg

class LabelSmoothingAttention(nn.Module):
    def __init__(self, d_model, num_heads, label_smoothing=0.1):
        super(LabelSmoothingAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.label_smoothing = label_smoothing
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply label smoothing to attention weights
        if self.training:
            uniform_weight = self.label_smoothing / attention_weights.size(-1)
            attention_weights = (1 - self.label_smoothing) * attention_weights + uniform_weight
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output

# Example usage with regularization
regularized_attn = AttentionRegularization(d_model=512, num_heads=8, dropout=0.1)
query = torch.randn(32, 20, 512)
key = torch.randn(32, 20, 512)
value = torch.randn(32, 20, 512)

output, attn_weights, entropy_reg = regularized_attn(query, key, value)
print(f"Regularized attention output: {output.shape}")
print(f"Entropy regularization: {entropy_reg.item():.4f}")
print(f"Attention weights shape: {attn_weights.shape}")
```

## 16. How do you implement attention interpretability and visualization?

**Answer:**
```python
import matplotlib.pyplot as plt
import seaborn as sns

class InterpretableAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super(InterpretableAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output, attention_weights

def visualize_attention(attention_weights, source_tokens, target_tokens, title="Attention Weights"):
    """
    Visualize attention weights as a heatmap
    """
    # Convert to numpy for plotting
    if isinstance(attention_weights, torch.Tensor):
        weights = attention_weights[0].cpu().numpy()  # Take first item in batch
    
    # Create heatmap
    plt.figure(figsize=(12, 8))
    sns.heatmap(
        weights, 
        xticklabels=source_tokens, 
        yticklabels=target_tokens,
        cmap='viridis',
        cbar=True
    )
    plt.title(title)
    plt.xlabel('Source Tokens')
    plt.ylabel('Target Tokens')
    plt.xticks(rotation=45, ha='right')
    plt.yticks(rotation=0)
    plt.tight_layout()
    plt.show()

def analyze_attention_patterns(attention_weights):
    """
    Analyze attention patterns and provide insights
    """
    # Convert to numpy for analysis
    if isinstance(attention_weights, torch.Tensor):
        weights = attention_weights.cpu().numpy()
    
    # Calculate statistics
    max_attention = np.max(weights, axis=-1)  # Most attended position per query
    attention_entropy = -np.sum(weights * np.log(weights + 1e-8), axis=-1)  # Uncertainty measure
    
    # Diagonal attention (for self-attention) - how much each position attends to itself
    diagonal_attention = np.diagonal(weights, axis1=-2, axis2=-1)
    
    analysis = {
        'max_attention_positions': np.argmax(weights, axis=-1),
        'attention_entropy': attention_entropy,
        'diagonal_attention': diagonal_attention,
        'attention_sparsity': 1 - np.count_nonzero(weights > 0.01) / weights.size,
        'attention_concentration': np.max(weights, axis=-1) / np.sum(weights, axis=-1)
    }
    
    return analysis

class AttentionAnalyzer:
    def __init__(self, model):
        self.model = model
        self.attention_weights = []
        self.hook_handles = []
    
    def register_hooks(self):
        """Register forward hooks to capture attention weights"""
        def hook_fn(module, input, output):
            if len(output) == 2:  # Output includes both result and attention weights
                self.attention_weights.append(output[1].detach())
        
        # Register hooks on all attention modules in the model
        for name, module in self.model.named_modules():
            if isinstance(module, (InterpretableAttention,)):
                handle = module.register_forward_hook(hook_fn)
                self.hook_handles.append(handle)
    
    def remove_hooks(self):
        """Remove all registered hooks"""
        for handle in self.hook_handles:
            handle.remove()
        self.hook_handles.clear()
        self.attention_weights.clear()
    
    def get_attention_weights(self):
        """Return captured attention weights"""
        return self.attention_weights.copy()

# Example usage
def demonstrate_attention_interpretability():
    # Create attention model
    attention_model = InterpretableAttention(d_model=128, num_heads=8)
    
    # Sample inputs
    query = torch.randn(1, 10, 128)  # (batch, seq_len, d_model)
    key = torch.randn(1, 10, 128)
    value = torch.randn(1, 10, 128)
    
    # Forward pass
    output, attention_weights = attention_model(query, key, value)
    
    # Sample tokens for visualization
    source_tokens = [f"src_{i}" for i in range(10)]
    target_tokens = [f"tgt_{i}" for i in range(10)]
    
    print("Attention analysis:")
    print(f"Output shape: {output.shape}")
    print(f"Attention weights shape: {attention_weights.shape}")
    
    # Analyze attention patterns
    analysis = analyze_attention_patterns(attention_weights)
    print(f"Attention entropy: {np.mean(analysis['attention_entropy']):.4f}")
    print(f"Attention sparsity: {analysis['attention_sparsity']:.4f}")
    
    # Visualize attention (commented out to avoid display issues in console)
    # visualize_attention(attention_weights, source_tokens, target_tokens)
    
    print("Attention interpretability tools ready")

# Transformer attention visualization
def plot_transformer_attention(attention_weights, layer_idx=0, head_idx=0):
    """
    Plot attention weights for a specific layer and head in a transformer
    """
    # attention_weights: (batch, num_heads, seq_len, seq_len)
    if isinstance(attention_weights, (list, tuple)):
        # If we have a list of attention weights from different layers
        weights = attention_weights[layer_idx][0, head_idx]  # Take first batch, specific layer and head
    else:
        weights = attention_weights[0, head_idx]  # Take first batch, specific head
    
    weights = weights.cpu().numpy()
    
    plt.figure(figsize=(10, 8))
    sns.heatmap(
        weights,
        xticklabels=[f"pos_{i}" for i in range(weights.shape[1])],
        yticklabels=[f"pos_{i}" for i in range(weights.shape[0])],
        cmap='Blues',
        annot=False
    )
    plt.title(f'Attention Weights - Layer {layer_idx}, Head {head_idx}')
    plt.tight_layout()
    plt.show()

# Run demonstration
demonstrate_attention_interpretability()
```

## 17. How do you implement cross-attention mechanisms?

**Answer:**
```python
class CrossAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(CrossAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        assert d_model % num_heads == 0
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        """
        query: (batch_size, query_len, d_model) - from target sequence
        key: (batch_size, key_len, d_model) - from source sequence  
        value: (batch_size, value_len, d_model) - from source sequence
        """
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores (query attends to key/value)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention to values
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output, attention_weights

class DualAttention(nn.Module):
    """
    Combines self-attention and cross-attention
    """
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(DualAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        
        # Self-attention for source and target
        self.self_attn_src = MultiHeadAttention(d_model, num_heads, dropout)
        self.self_attn_tgt = MultiHeadAttention(d_model, num_heads, dropout)
        
        # Cross-attention (target attends to source)
        self.cross_attn = CrossAttention(d_model, num_heads, dropout)
        
        # Layer normalization
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        
        # Feed-forward networks
        self.ffn1 = nn.Sequential(
            nn.Linear(d_model, d_model * 4),
            nn.ReLU(),
            nn.Linear(d_model * 4, d_model),
            nn.Dropout(dropout)
        )
        self.ffn2 = nn.Sequential(
            nn.Linear(d_model, d_model * 4),
            nn.ReLU(),
            nn.Linear(d_model * 4, d_model),
            nn.Dropout(dropout)
        )
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, src, tgt, src_mask=None, tgt_mask=None, cross_mask=None):
        """
        src: (batch_size, src_len, d_model)
        tgt: (batch_size, tgt_len, d_model)
        """
        # Self-attention on source
        src_self, _ = self.self_attn_src(src, src, src, src_mask)
        src = self.norm1(src + self.dropout(src_self))
        
        # Self-attention on target
        tgt_self, _ = self.self_attn_tgt(tgt, tgt, tgt, tgt_mask)
        tgt = self.norm2(tgt + self.dropout(tgt_self))
        
        # Cross-attention (target attends to source)
        cross_output, cross_weights = self.cross_attn(tgt, src, src, cross_mask)
        tgt = self.norm3(tgt + self.dropout(cross_output))
        
        # Feed-forward networks
        src = self.norm1(src + self.dropout(self.ffn1(src)))
        tgt = self.norm2(tgt + self.dropout(self.ffn2(tgt)))
        
        return src, tgt, cross_weights

class ImageTextCrossAttention(nn.Module):
    """
    Cross-attention between image features and text embeddings
    """
    def __init__(self, image_dim, text_dim, hidden_dim):
        super(ImageTextCrossAttention, self).__init__()
        self.image_to_hidden = nn.Linear(image_dim, hidden_dim)
        self.text_to_hidden = nn.Linear(text_dim, hidden_dim)
        self.attention = CrossAttention(hidden_dim, num_heads=8)
        
    def forward(self, image_features, text_embeddings):
        """
        image_features: (batch_size, num_regions, image_dim)
        text_embeddings: (batch_size, seq_len, text_dim)
        """
        # Project to common hidden dimension
        img_proj = F.relu(self.image_to_hidden(image_features))
        text_proj = F.relu(self.text_to_hidden(text_embeddings))
        
        # Cross-attention: text attends to image
        text_attended, text_weights = self.attention(
            query=text_proj, 
            key=img_proj, 
            value=img_proj
        )
        
        # Cross-attention: image attends to text
        img_attended, img_weights = self.attention(
            query=img_proj, 
            key=text_proj, 
            value=text_proj
        )
        
        return text_attended, img_attended, text_weights, img_weights

# Example usage
cross_attn = CrossAttention(d_model=512, num_heads=8)
query = torch.randn(32, 20, 512)  # Target sequence
key = torch.randn(32, 30, 512)    # Source sequence
value = torch.randn(32, 30, 512)  # Source sequence

output, weights = cross_attn(query, key, value)
print(f"Cross-attention output: {output.shape}")
print(f"Cross-attention weights: {weights.shape}")

# Dual attention example
dual_attn = DualAttention(d_model=256, num_heads=8)
src_seq = torch.randn(16, 25, 256)
tgt_seq = torch.randn(16, 20, 256)

src_out, tgt_out, cross_weights = dual_attn(src_seq, tgt_seq)
print(f"Dual attention - Source output: {src_out.shape}, Target output: {tgt_out.shape}")
print(f"Cross attention weights: {cross_weights.shape}")
```

## 18. How do you implement attention with relative positional encoding?

**Answer:**
```python
class RelativePositionAttention(nn.Module):
    """
    Attention with relative positional encoding
    """
    def __init__(self, d_model, num_heads, max_relative_position=16):
        super(RelativePositionAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.max_relative_position = max_relative_position
        
        # Linear transformations
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Relative position embeddings
        vocab_size = max_relative_position * 2 + 1
        self.relative_k_embed = nn.Embedding(vocab_size, self.d_k)
        self.relative_v_embed = nn.Embedding(vocab_size, self.d_k)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Create relative position embeddings
        seq_len = Q.size(2)
        positions = torch.arange(seq_len, dtype=torch.long, device=query.device)
        relative_positions = positions.unsqueeze(1) - positions.unsqueeze(0)  # (seq_len, seq_len)
        
        # Clamp relative positions to the allowed range
        relative_positions = torch.clamp(
            relative_positions, 
            -self.max_relative_position, 
            self.max_relative_position
        )
        
        # Shift to positive indices
        relative_positions = relative_positions + self.max_relative_position
        
        # Get relative position embeddings
        relative_k = self.relative_k_embed(relative_positions)  # (seq_len, seq_len, d_k)
        relative_v = self.relative_v_embed(relative_positions)  # (seq_len, seq_len, d_k)
        
        # Calculate attention scores with relative positions
        # Standard attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Add relative position attention for keys
        relative_k = relative_k.unsqueeze(0).unsqueeze(0)  # (1, 1, seq_len, seq_len, d_k)
        relative_k = relative_k.expand(batch_size, self.num_heads, -1, -1, -1)
        Q_expanded = Q.unsqueeze(3).expand(-1, -1, -1, seq_len, -1)
        relative_attn_k = torch.sum(Q_expanded * relative_k, dim=-1)  # (batch, num_heads, seq_len, seq_len)
        scores = scores + relative_attn_k
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Calculate attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention to values
        output = torch.matmul(attention_weights, V)
        
        # Add relative position attention for values
        relative_v = relative_v.unsqueeze(0).unsqueeze(0)  # (1, 1, seq_len, seq_len, d_k)
        relative_v = relative_v.expand(batch_size, self.num_heads, -1, -1, -1)
        rel_attn_output = torch.matmul(attention_weights.unsqueeze(-1), relative_v).squeeze(-2)
        
        # Combine outputs
        output = output + rel_attn_output
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output, attention_weights

class RelativePositionTransformer(nn.Module):
    def __init__(self, d_model, num_heads, num_layers, max_relative_position=16):
        super(RelativePositionTransformer, self).__init__()
        self.layers = nn.ModuleList([
            nn.TransformerEncoderLayer(
                d_model=d_model,
                nhead=num_heads,
                batch_first=True
            ) for _ in range(num_layers)
        ])
        
        # Add relative position attention layers
        self.relative_attention_layers = nn.ModuleList([
            RelativePositionAttention(d_model, num_heads, max_relative_position)
            for _ in range(num_layers)
        ])
        
        self.norm = nn.LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        for layer, rel_attn in zip(self.layers, self.relative_attention_layers):
            # Apply standard transformer layer
            x = layer(x, src_mask=mask)
            
            # Apply relative position attention
            rel_output, _ = rel_attn(x, x, x, mask)
            x = self.norm(x + rel_output)
        
        return x

# Example usage
rel_pos_attn = RelativePositionAttention(d_model=256, num_heads=8, max_relative_position=10)
query = torch.randn(32, 20, 256)
key = torch.randn(32, 20, 256)
value = torch.randn(32, 20, 256)

output, weights = rel_pos_attn(query, key, value)
print(f"Relative position attention output: {output.shape}")
print(f"Attention weights: {weights.shape}")

# Relative position transformer
rel_transformer = RelativePositionTransformer(
    d_model=256, 
    num_heads=8, 
    num_layers=2, 
    max_relative_position=8
)

transformer_output = rel_transformer(query)
print(f"Relative position transformer output: {transformer_output.shape}")
```

## 19. How do you implement attention with routing mechanisms?

**Answer:**
```python
class RoutingAttention(nn.Module):
    """
    Attention with routing mechanisms - similar to mixture of attention heads
    """
    def __init__(self, d_model, num_heads, num_experts=4, dropout=0.1):
        super(RoutingAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.num_experts = num_experts
        self.d_k = d_model // num_heads
        
        # Multiple attention "experts"
        self.experts = nn.ModuleList([
            MultiHeadAttention(d_model, num_heads, dropout)
            for _ in range(num_experts)
        ])
        
        # Router network
        self.router = nn.Linear(d_model, num_experts)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size, seq_len = query.size(0), query.size(1)
        
        # Get routing weights
        routing_input = torch.mean(query, dim=1)  # Average over sequence length
        routing_weights = F.softmax(self.router(routing_input), dim=-1)  # (batch, num_experts)
        
        # Compute outputs from all experts
        expert_outputs = []
        for expert in self.experts:
            output, _ = expert(query, key, value, mask)
            expert_outputs.append(output.unsqueeze(0))  # Add expert dimension
        
        # Stack expert outputs: (num_experts, batch, seq_len, d_model)
        expert_outputs = torch.cat(expert_outputs, dim=0)
        
        # Route outputs based on routing weights
        # routing_weights: (batch, num_experts)
        # expert_outputs: (num_experts, batch, seq_len, d_model)
        routing_weights = routing_weights.transpose(0, 1).unsqueeze(-1).unsqueeze(-1)
        # routing_weights: (num_experts, batch, 1, 1)
        
        routed_output = torch.sum(expert_outputs * routing_weights, dim=0)  # (batch, seq_len, d_model)
        
        return routed_output, routing_weights.squeeze(-1).squeeze(-1)

class GatingMechanismAttention(nn.Module):
    """
    Attention with gating mechanisms
    """
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(GatingMechanismAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        
        # Two different attention mechanisms
        self.attention1 = MultiHeadAttention(d_model, num_heads, dropout)
        self.attention2 = MultiHeadAttention(d_model, num_heads, dropout)
        
        # Gating network
        self.gate = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.ReLU(),
            nn.Linear(d_model // 2, 2),
            nn.Softmax(dim=-1)
        )
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        # Compute both attention outputs
        output1, _ = self.attention1(query, key, value, mask)
        output2, _ = self.attention2(query, key, value, mask)
        
        # Compute gate weights
        gate_input = torch.mean(query, dim=1)  # (batch, d_model)
        gate_weights = self.gate(gate_input).unsqueeze(1)  # (batch, 1, 2)
        
        # Apply gating
        combined_output = gate_weights[:, :, 0:1] * output1 + gate_weights[:, :, 1:2] * output2
        
        return combined_output, gate_weights

class DynamicAttention(nn.Module):
    """
    Dynamic attention that adapts based on input characteristics
    """
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(DynamicAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Different attention mechanisms for different scenarios
        self.local_attention = MultiHeadAttention(d_model, num_heads, dropout)
        self.global_attention = MultiHeadAttention(d_model, num_heads, dropout)
        
        # Dynamic selector network
        self.selector = nn.Sequential(
            nn.Linear(d_model, d_model // 4),
            nn.ReLU(),
            nn.Linear(d_model // 4, 2),
            nn.Softmax(dim=-1)
        )
        
        # Convolution for local pattern detection
        self.local_pattern_detector = nn.Conv1d(d_model, d_model, kernel_size=3, padding=1)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size, seq_len = query.size(0), query.size(1)
        
        # Compute local and global attention
        local_output, _ = self.local_attention(query, key, value, mask)
        global_output, _ = self.global_attention(query, key, value, mask)
        
        # Detect local patterns
        local_patterns = F.relu(self.local_pattern_detector(query.transpose(1, 2))).transpose(1, 2)
        
        # Average over sequence to get a representation for selection
        avg_representation = torch.mean(local_patterns, dim=1)  # (batch, d_model)
        
        # Compute selection weights
        selection_weights = self.selector(avg_representation).unsqueeze(1)  # (batch, 1, 2)
        
        # Combine outputs based on selection
        dynamic_output = selection_weights[:, :, 0:1] * local_output + selection_weights[:, :, 1:2] * global_output
        
        return dynamic_output, selection_weights

# Example usage
routing_attn = RoutingAttention(d_model=256, num_heads=8, num_experts=4)
query = torch.randn(16, 20, 256)
key = torch.randn(16, 20, 256)
value = torch.randn(16, 20, 256)

output, routing_weights = routing_attn(query, key, value)
print(f"Routing attention output: {output.shape}")
print(f"Routing weights: {routing_weights.shape}")

gating_attn = GatingMechanismAttention(d_model=256, num_heads=8)
gated_output, gate_weights = gating_attn(query, key, value)
print(f"Gating attention output: {gated_output.shape}")
print(f"Gate weights: {gate_weights.shape}")

dynamic_attn = DynamicAttention(d_model=256, num_heads=8)
dynamic_output, selection_weights = dynamic_attn(query, key, value)
print(f"Dynamic attention output: {dynamic_output.shape}")
print(f"Selection weights: {selection_weights.shape}")
```

## 20. How do you implement attention with neural architecture search (NAS)?

**Answer:**
```python
class NASAttention(nn.Module):
    """
    Attention mechanism with differentiable architecture search
    """
    def __init__(self, d_model, max_num_heads=8, search_space=None):
        super(NASAttention, self).__init__()
        self.d_model = d_model
        self.max_num_heads = max_num_heads
        
        if search_space is None:
            search_space = ['scaled_dot_product', 'additive', 'multiplicative']
        
        self.search_space = search_space
        self.num_operations = len(search_space)
        
        # Different attention implementations
        self.attention_ops = nn.ModuleList([
            self._create_attention_op(op, d_model) 
            for op in search_space
        ])
        
        # Architecture parameters (learnable weights for each operation)
        self.arch_params = nn.Parameter(torch.zeros(self.num_operations))
        
    def _create_attention_op(self, op_type, d_model):
        if op_type == 'scaled_dot_product':
            return MultiHeadAttention(d_model, num_heads=min(self.max_num_heads, 8), dropout=0.1)
        elif op_type == 'additive':
            return BahdanauAttention(d_model)
        elif op_type == 'multiplicative':
            return LuongAttention(d_model, method='general')
        else:
            raise ValueError(f"Unknown attention operation: {op_type}")
    
    def forward(self, query, key, value, mask=None, temperature=1.0):
        # Apply each attention operation
        outputs = []
        for op in self.attention_ops:
            if hasattr(op, 'forward'):
                # Different operations may have different signatures
                if isinstance(op, MultiHeadAttention):
                    output, _ = op(query, key, value, mask)
                elif isinstance(op, (BahdanauAttention, LuongAttention)):
                    # These operate differently, need to adapt
                    # For this example, we'll use a placeholder approach
                    output = torch.randn_like(query)  # Placeholder
                else:
                    output = torch.randn_like(query)  # Placeholder
                outputs.append(output)
        
        # Apply Gumbel-Softmax for differentiable architecture selection
        arch_weights = F.gumbel_softmax(self.arch_params.unsqueeze(0), tau=temperature, hard=False)
        
        # Weighted combination of outputs
        combined_output = torch.stack(outputs, dim=0)  # (num_ops, batch, seq_len, d_model)
        arch_weights = arch_weights.transpose(0, 1).unsqueeze(-1).unsqueeze(-1)  # (num_ops, 1, 1, 1)
        
        final_output = torch.sum(combined_output * arch_weights, dim=0)  # (batch, seq_len, d_model)
        
        return final_output, arch_weights.squeeze().detach().cpu().numpy()

class DifferentiableAttention(nn.Module):
    """
    Attention with differentiable head selection
    """
    def __init__(self, d_model, max_heads=8):
        super(DifferentiableAttention, self).__init__()
        self.d_model = d_model
        self.max_heads = max_heads
        self.d_k = d_model // max_heads
        
        # Linear transformations (for maximum possible heads)
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Head selection weights (learnable)
        self.head_weights = nn.Parameter(torch.ones(max_heads))
        
    def forward(self, query, key, value, mask=None, temperature=1.0):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.max_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.max_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.max_heads, self.d_k).transpose(1, 2)
        
        # Apply head selection using Gumbel-Softmax
        head_weights = F.gumbel_softmax(self.head_weights.unsqueeze(0), tau=temperature, hard=False)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply head selection to attention weights
        # head_weights: (1, max_heads) -> (1, max_heads, 1, 1)
        head_weights = head_weights.unsqueeze(-1).unsqueeze(-1)
        weighted_attention = attention_weights * head_weights
        
        # Apply attention
        output = torch.matmul(weighted_attention, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output, head_weights.squeeze().detach().cpu().numpy()

class EvolutionaryAttention(nn.Module):
    """
    Attention mechanism that can evolve its architecture
    """
    def __init__(self, d_model, initial_configs=None):
        super(EvolutionaryAttention, self).__init__()
        self.d_model = d_model
        
        if initial_configs is None:
            # Default configurations to start with
            initial_configs = [
                {'num_heads': 4, 'dropout': 0.1, 'activation': 'relu'},
                {'num_heads': 8, 'dropout': 0.1, 'activation': 'relu'},
                {'num_heads': 1, 'dropout': 0.2, 'activation': 'gelu'}
            ]
        
        self.configs = initial_configs
        self.num_configs = len(initial_configs)
        
        # Create attention modules for each configuration
        self.attention_modules = nn.ModuleList([
            self._create_attention_from_config(config) 
            for config in initial_configs
        ])
        
        # Performance tracking for evolution
        self.performance_scores = nn.Parameter(torch.zeros(self.num_configs), requires_grad=False)
        
    def _create_attention_from_config(self, config):
        num_heads = config.get('num_heads', 8)
        dropout = config.get('dropout', 0.1)
        activation = config.get('activation', 'relu')
        
        # Create attention based on configuration
        attention = MultiHeadAttention(
            self.d_model, 
            num_heads=num_heads, 
            dropout=dropout
        )
        
        # Add activation function if specified
        activation_fn = nn.ReLU() if activation == 'relu' else nn.GELU()
        return nn.Sequential(attention, activation_fn)
    
    def forward(self, query, key, value, mask=None, selection_method='best'):
        outputs = []
        
        # Get output from each configuration
        for module in self.attention_modules:
            # The first element in the sequential is the attention module
            if hasattr(module[0], 'forward'):
                output, _ = module[0](query, key, value, mask)
                output = module[1](output)  # Apply activation
                outputs.append(output)
        
        if selection_method == 'best':
            # Select based on performance scores
            best_idx = torch.argmax(self.performance_scores)
            return outputs[best_idx], best_idx
        elif selection_method == 'weighted':
            # Weighted combination based on scores
            weights = F.softmax(self.performance_scores, dim=0)
            combined_output = torch.stack(outputs, dim=0)  # (num_configs, batch, seq_len, d_model)
            weights = weights.unsqueeze(1).unsqueeze(1).unsqueeze(1)  # (num_configs, 1, 1, 1)
            final_output = torch.sum(combined_output * weights, dim=0)
            return final_output, weights.squeeze().detach().cpu().numpy()
        else:
            # Return all outputs and let caller decide
            return outputs, self.performance_scores.detach().cpu().numpy()
    
    def update_performance(self, config_idx, score):
        """Update performance score for a configuration"""
        with torch.no_grad():
            self.performance_scores[config_idx] = score

# Example usage
nas_attn = NASAttention(d_model=128, max_num_heads=4)
query = torch.randn(8, 15, 128)
key = torch.randn(8, 15, 128)
value = torch.randn(8, 15, 128)

output, arch_weights = nas_attn(query, key, value, temperature=0.5)
print(f"NAS attention output: {output.shape}")
print(f"Architecture weights: {arch_weights}")

diff_attn = DifferentiableAttention(d_model=128, max_heads=4)
diff_output, head_weights = diff_attn(query, key, value, temperature=0.5)
print(f"Differentiable attention output: {diff_output.shape}")
print(f"Head weights: {head_weights}")

evol_attn = EvolutionaryAttention(d_model=128)
evol_output, selection_info = evol_attn(query, key, value, selection_method='weighted')
print(f"Evolutionary attention output: {evol_output.shape}")
print(f"Selection info: {selection_info}")
```

## 21. How do you implement attention with memory-augmented neural networks?

**Answer:**
```python
class MemoryAugmentedAttention(nn.Module):
    """
    Attention mechanism with external memory
    """
    def __init__(self, d_model, memory_size=100, memory_dim=64):
        super(MemoryAugmentedAttention, self).__init__()
        self.d_model = d_model
        self.memory_size = memory_size
        self.memory_dim = memory_dim
        
        # Memory matrix (initialized randomly)
        self.memory = nn.Parameter(torch.randn(memory_size, memory_dim))
        
        # Controller networks
        self.key_generator = nn.Linear(d_model, memory_dim)
        self.value_generator = nn.Linear(d_model, memory_dim)
        self.read_projection = nn.Linear(d_model + memory_dim, d_model)
        
        # Attention for memory addressing
        self.memory_attention = MultiHeadAttention(memory_dim, num_heads=4, dropout=0.1)
        
    def forward(self, query, key, value, mask=None, external_memory=None):
        batch_size = query.size(0)
        
        # Use provided external memory or internal memory
        if external_memory is not None:
            memory_matrix = external_memory
        else:
            memory_matrix = self.memory.unsqueeze(0).expand(batch_size, -1, -1)
        
        # Generate read and write keys from input
        query_keys = self.key_generator(query)  # (batch, seq_len, memory_dim)
        
        # Calculate attention between input and memory
        scores = torch.bmm(query_keys, memory_matrix.transpose(1, 2))  # (batch, seq_len, memory_size)
        attention_weights = F.softmax(scores, dim=-1)  # (batch, seq_len, memory_size)
        
        # Read from memory
        read_memory = torch.bmm(attention_weights, memory_matrix)  # (batch, seq_len, memory_dim)
        
        # Combine input with read memory
        combined_input = torch.cat([query, read_memory], dim=-1)
        output = self.read_projection(combined_input)
        
        # Update memory based on current input
        write_values = self.value_generator(query)  # (batch, seq_len, memory_dim)
        
        # For simplicity, update first k memory slots based on input
        update_indices = torch.arange(min(query.size(1), self.memory_size)).to(query.device)
        updated_memory = memory_matrix.clone()
        for i in range(batch_size):
            for j, idx in enumerate(update_indices):
                if j < write_values.size(1):
                    updated_memory[i, idx] = write_values[i, j]
        
        return output, attention_weights, updated_memory

class NeuralTuringMachineAttention(nn.Module):
    """
    Attention-based Neural Turing Machine
    """
    def __init__(self, input_size, controller_size, memory_size=128, memory_dim=20):
        super(NeuralTuringMachineAttention, self).__init__()
        
        self.controller_size = controller_size
        self.memory_size = memory_size
        self.memory_dim = memory_dim
        
        # Controller LSTM
        self.controller = nn.LSTMCell(input_size + memory_dim, controller_size)
        
        # Memory attention mechanism
        self.write_head = nn.Linear(controller_size, memory_dim)
        self.read_head = nn.Linear(controller_size, memory_dim)
        self.location_weights = nn.Linear(controller_size, memory_size)
        
        # Output projection
        self.output = nn.Linear(controller_size + memory_dim, input_size)
        
    def forward(self, input_sequences, initial_memory=None):
        """
        input_sequences: (batch_size, seq_len, input_size)
        """
        batch_size, seq_len, input_size = input_sequences.size()
        
        # Initialize memory
        if initial_memory is None:
            memory = torch.randn(batch_size, self.memory_size, self.memory_dim).to(input_sequences.device)
        else:
            memory = initial_memory
        
        # Initialize controller states
        hx = torch.zeros(batch_size, self.controller_size).to(input_sequences.device)
        cx = torch.zeros(batch_size, self.controller_size).to(input_sequences.device)
        
        outputs = []
        
        for t in range(seq_len):
            current_input = input_sequences[:, t, :]  # (batch, input_size)
            
            # Get read weights from controller
            read_weights = F.softmax(self.location_weights(hx), dim=1)  # (batch, memory_size)
            
            # Read from memory
            read_content = torch.bmm(read_weights.unsqueeze(1), memory).squeeze(1)  # (batch, memory_dim)
            
            # Controller input is concatenation of input and read content
            controller_input = torch.cat([current_input, read_content], dim=1)
            
            # Update controller
            hx, cx = self.controller(controller_input, (hx, cx))
            
            # Generate write content
            write_content = torch.tanh(self.write_head(hx))  # (batch, memory_dim)
            
            # Generate write weights
            write_weights = F.softmax(self.location_weights(hx), dim=1)  # (batch, memory_size)
            
            # Write to memory
            write_update = torch.bmm(write_weights.unsqueeze(2), write_content.unsqueeze(1))  # (batch, memory_size, memory_dim)
            memory = memory + write_update
            
            # Generate output
            output_input = torch.cat([hx, read_content], dim=1)
            output = self.output(output_input)
            outputs.append(output)
        
        outputs = torch.stack(outputs, dim=1)  # (batch, seq_len, input_size)
        
        return outputs, memory

class DifferentiableNeuralComputer(nn.Module):
    """
    Differentiable Neural Computer with attention-based memory access
    """
    def __init__(self, input_size, controller_size, memory_size=100, memory_dim=64):
        super(DifferentiableNeuralComputer, self).__init__()
        
        self.controller_size = controller_size
        self.memory_size = memory_size
        self.memory_dim = memory_dim
        
        # Controller network
        self.controller = nn.LSTM(input_size, controller_size, batch_first=True)
        
        # Interface between controller and memory
        self.interface = nn.Linear(controller_size, 
                                 3 * memory_dim +  # key, strength, key_beta
                                 3 * 1 +           # shift, gamma, beta
                                 memory_size + 1   # write weights, free gates
                                 )
        
        # Output layer
        self.output = nn.Linear(controller_size + memory_dim, input_size)
        
    def content_addressing(self, memory, keys, strengths):
        """
        Content-based addressing mechanism
        """
        # Normalize memory and keys
        norm_memory = F.normalize(memory, p=2, dim=-1)
        norm_keys = F.normalize(keys, p=2, dim=-1)
        
        # Calculate similarity
        similarity = torch.bmm(norm_memory, norm_keys.transpose(-2, -1))  # (batch, memory_size, num_keys)
        
        # Apply strengths
        strengths = strengths.unsqueeze(1)  # (batch, 1, num_keys)
        weights = F.softmax(similarity * strengths, dim=1)  # (batch, memory_size, num_keys)
        
        return weights
    
    def forward(self, inputs):
        batch_size, seq_len, input_size = inputs.size()
        
        # Initialize memory
        memory = torch.randn(batch_size, self.memory_size, self.memory_dim).to(inputs.device)
        
        # Controller forward pass
        controller_output, _ = self.controller(inputs)  # (batch, seq_len, controller_size)
        
        outputs = []
        
        for t in range(seq_len):
            ctrl_out = controller_output[:, t, :]  # (batch, controller_size)
            
            # Generate interface parameters
            interface_params = self.interface(ctrl_out)  # (batch, interface_dim)
            
            # Parse interface parameters
            key = torch.tanh(interface_params[:, :self.memory_dim])  # Read key
            strength = torch.sigmoid(interface_params[:, self.memory_dim:self.memory_dim+1])  # Read strength
            write_key = torch.tanh(interface_params[:, self.memory_dim+1:2*self.memory_dim+1])  # Write key
            write_strength = torch.sigmoid(interface_params[:, 2*self.memory_dim+1:2*self.memory_dim+2])  # Write strength
            
            # Content addressing for reading
            read_weights = self.content_addressing(
                memory, key.unsqueeze(-1), strength
            ).squeeze(-1)  # (batch, memory_size)
            
            # Read from memory
            read_content = torch.bmm(read_weights.unsqueeze(1), memory).squeeze(1)  # (batch, memory_dim)
            
            # Generate output
            output_input = torch.cat([ctrl_out, read_content], dim=1)
            output = self.output(output_input)
            outputs.append(output)
        
        outputs = torch.stack(outputs, dim=1)  # (batch, seq_len, input_size)
        
        return outputs

# Example usage
mem_attn = MemoryAugmentedAttention(d_model=64, memory_size=50, memory_dim=32)
query = torch.randn(16, 20, 64)
key = torch.randn(16, 20, 64)
value = torch.randn(16, 20, 64)

output, attention_weights, updated_memory = mem_attn(query, key, value)
print(f"Memory-augmented attention output: {output.shape}")
print(f"Attention weights: {attention_weights.shape}")
print(f"Updated memory: {updated_memory.shape}")

# Example Neural Turing Machine
ntm = NeuralTuringMachineAttention(
    input_size=50,
    controller_size=100,
    memory_size=64,
    memory_dim=20
)

input_seq = torch.randn(8, 15, 50)
ntm_output, final_memory = ntm(input_seq)
print(f"NTM output: {ntm_output.shape}")
print(f"Final memory: {final_memory.shape}")
```

## 22. How do you implement attention with reinforcement learning?

**Answer:**
```python
import torch.nn.functional as F
from torch.distributions import Categorical

class RLAttention(nn.Module):
    """
    Attention mechanism trained with reinforcement learning
    """
    def __init__(self, d_model, num_heads, vocab_size=10000):
        super(RLAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # Standard attention components
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Policy network for attention selection
        self.policy_network = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.ReLU(),
            nn.Linear(d_model // 2, num_heads),  # One action per head
            nn.Softmax(dim=-1)
        )
        
        # Value network for baseline estimation
        self.value_network = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.ReLU(),
            nn.Linear(d_model // 2, 1)
        )
        
    def forward(self, query, key, value, mask=None, training=True):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Calculate attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Standard attention weights
        standard_attention = F.softmax(scores, dim=-1)
        
        # Get policy for head selection
        avg_query = torch.mean(query, dim=1)  # (batch, d_model)
        head_probs = self.policy_network(avg_query)  # (batch, num_heads)
        
        if training:
            # Sample heads based on policy during training
            m = Categorical(head_probs)
            head_actions = m.sample()  # (batch,)
            
            # Create attention mask based on selected heads
            head_mask = F.one_hot(head_actions, num_heads).float().unsqueeze(1).unsqueeze(-1)  # (batch, 1, 1, num_heads)
            
            # Apply head selection to attention weights
            # Expand head_mask to match attention dimensions
            head_mask = head_mask.expand(-1, self.num_heads, scores.size(-2), scores.size(-1))
            
            # Use selected heads
            selected_attention = standard_attention * head_mask[:, 0:1, :, :]  # Use first head mask for simplicity
            
            # Calculate value for baseline
            value = self.value_network(avg_query)  # (batch, 1)
            
            # Store for policy update
            self.saved_log_probs = m.log_prob(head_actions)  # (batch,)
            self.saved_values = value  # (batch, 1)
        else:
            # Use greedy selection during evaluation
            head_actions = torch.argmax(head_probs, dim=-1)  # (batch,)
            selected_attention = standard_attention
        
        # Apply attention
        output = torch.matmul(selected_attention, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output, head_actions if training else head_probs

class AttentionRewardFunction:
    """
    Reward function for attention mechanisms
    """
    @staticmethod
    def attention_concentration_reward(attention_weights, target_sparsity=0.1):
        """
        Reward for attention concentration (not too spread out)
        """
        # Calculate entropy of attention distribution
        entropy = -torch.sum(attention_weights * torch.log(attention_weights + 1e-8), dim=-1)
        
        # Higher entropy = more uniform attention = lower reward
        concentration_reward = -entropy
        
        return concentration_reward.mean()
    
    @staticmethod
    def attention_diversity_reward(attention_weights):
        """
        Reward for attention diversity across different heads
        """
        # Calculate the difference between attention patterns of different heads
        batch_size, num_heads, seq_len, _ = attention_weights.shape
        
        diversity_reward = 0
        for i in range(num_heads):
            for j in range(i + 1, num_heads):
                diff = torch.abs(attention_weights[:, i] - attention_weights[:, j])
                diversity_reward += torch.mean(diff)
        
        diversity_reward = diversity_reward / (num_heads * (num_heads - 1) / 2)
        return diversity_reward

class ActorCriticAttention(nn.Module):
    """
    Actor-Critic architecture for attention mechanism
    """
    def __init__(self, d_model, num_heads, action_space_size):
        super(ActorCriticAttention, self).__init__()
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.action_space_size = action_space_size
        
        # Actor (policy network) - selects which attention heads to use
        self.actor = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.ReLU(),
            nn.Linear(d_model // 2, action_space_size),
            nn.Softmax(dim=-1)
        )
        
        # Critic (value network) - estimates value of attention configuration
        self.critic = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.ReLU(),
            nn.Linear(d_model // 2, 1)
        )
        
        # Standard attention mechanism
        self.attention = MultiHeadAttention(d_model, num_heads, dropout=0.1)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Get state representation (use query as state)
        state = torch.mean(query, dim=1)  # (batch, d_model)
        
        # Actor: get action probabilities
        action_probs = self.actor(state)  # (batch, action_space_size)
        action_dist = Categorical(action_probs)
        action = action_dist.sample()  # (batch,)
        
        # Critic: get state value
        state_value = self.critic(state)  # (batch, 1)
        
        # Apply attention (for now, use standard attention)
        attended_output, attention_weights = self.attention(query, key, value, mask)
        
        return attended_output, action, action_probs, state_value, attention_weights
    
    def get_action_probabilities(self, state):
        """Get action probabilities for a given state"""
        return self.actor(state)

class CurriculumLearningAttention(nn.Module):
    """
    Attention with curriculum learning - gradually increase difficulty
    """
    def __init__(self, d_model, num_heads, max_difficulty=5):
        super(CurriculumLearningAttention, self).__init__()
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.max_difficulty = max_difficulty
        
        # Different attention mechanisms for different difficulties
        self.attention_mechanisms = nn.ModuleList([
            MultiHeadAttention(d_model, min(num_heads, 2**(i+1)), dropout=0.1)
            for i in range(max_difficulty)
        ])
        
        # Difficulty level (can be updated during training)
        self.difficulty_level = 0
        
    def forward(self, query, key, value, mask=None):
        # Use attention mechanism based on current difficulty
        attention_idx = min(self.difficulty_level, len(self.attention_mechanisms) - 1)
        attention_mechanism = self.attention_mechanisms[attention_idx]
        
        output, attention_weights = attention_mechanism(query, key, value, mask)
        
        return output, attention_weights
    
    def increase_difficulty(self):
        """Gradually increase the difficulty level"""
        if self.difficulty_level < self.max_difficulty - 1:
            self.difficulty_level += 1
            print(f"Increased attention difficulty to level {self.difficulty_level + 1}")

# Example usage
rl_attn = RLAttention(d_model=128, num_heads=8)
query = torch.randn(16, 20, 128)
key = torch.randn(16, 20, 128)
value = torch.randn(16, 20, 128)

output, head_actions = rl_attn(query, key, value, training=True)
print(f"RL Attention output: {output.shape}")
print(f"Head actions: {head_actions.shape}")

# Example reward calculation
standard_attn = MultiHeadAttention(128, 8, dropout=0.1)
_, attn_weights = standard_attn(query, key, value)
concentration_reward = AttentionRewardFunction.attention_concentration_reward(attn_weights)
diversity_reward = AttentionRewardFunction.attention_diversity_reward(attn_weights)

print(f"Concentration reward: {concentration_reward.item():.4f}")
print(f"Diversity reward: {diversity_reward.item():.4f}")

# Actor-Critic example
ac_attn = ActorCriticAttention(d_model=128, num_heads=8, action_space_size=4)
ac_output, action, action_probs, state_value, attn_weights = ac_attn(query, key, value)
print(f"Actor-Critic output: {ac_output.shape}")
print(f"Action: {action.shape}, State value: {state_value.shape}")

# Curriculum learning example
curriculum_attn = CurriculumLearningAttention(d_model=64, num_heads=8, max_difficulty=3)
curriculum_output, curriculum_weights = curriculum_attn(query, key, value)
print(f"Curriculum attention output: {curriculum_output.shape}")

# Increase difficulty
curriculum_attn.increase_difficulty()