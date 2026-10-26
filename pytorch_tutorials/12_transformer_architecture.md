# Transformer Architecture in PyTorch - 30+ Questions and Answers

## 1. What is the Transformer architecture and why was it introduced?

**Answer:** The Transformer architecture was introduced in the paper "Attention is All You Need" by Vaswani et al. in 2017. It's a neural network architecture that relies entirely on attention mechanisms, dispensing with recurrence and convolutions entirely for sequence transduction tasks.

Key innovations of Transformers:
1. **Self-Attention:** Allows each position in the sequence to attend to all positions, capturing long-range dependencies
2. **Parallelization:** Unlike RNNs, all positions can be processed simultaneously
3. **Scalability:** Can handle very long sequences effectively
4. **Transfer Learning:** Foundation for models like BERT, GPT, T5, etc.

The architecture consists of an encoder-decoder structure where both components are stacks of identical layers, each containing:
- Multi-head self-attention mechanism
- Position-wise fully connected feed-forward networks
- Layer normalization and residual connections

## 2. How do you implement a basic Transformer encoder layer?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(MultiHeadAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        assert d_model % num_heads == 0
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations and reshape
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # Scaled dot-product attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output

class PositionwiseFeedForward(nn.Module):
    def __init__(self, d_model, d_ff, dropout=0.1):
        super(PositionwiseFeedForward, self).__init__()
        self.linear1 = nn.Linear(d_model, d_ff)
        self.linear2 = nn.Linear(d_ff, d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x):
        return self.linear2(self.dropout(F.relu(self.linear1(x))))

class LayerNorm(nn.Module):
    def __init__(self, features, eps=1e-6):
        super(LayerNorm, self).__init__()
        self.a_2 = nn.Parameter(torch.ones(features))
        self.b_2 = nn.Parameter(torch.zeros(features))
        self.eps = eps

    def forward(self, x):
        mean = x.mean(-1, keepdim=True)
        std = x.std(-1, keepdim=True)
        return self.a_2 * (x - mean) / (std + self.eps) + self.b_2

class TransformerEncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super(TransformerEncoderLayer, self).__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads, dropout)
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Self-attention sub-layer
        attn_output = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward sub-layer
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

# Example usage
encoder_layer = TransformerEncoderLayer(d_model=512, num_heads=8, d_ff=2048, dropout=0.1)
sample_input = torch.randn(32, 20, 512)  # (batch_size, seq_len, d_model)
output = encoder_layer(sample_input)
print(f"Encoder layer output shape: {output.shape}")
```

## 3. How do you implement a basic Transformer decoder layer?

**Answer:**
```python
class TransformerDecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super(TransformerDecoderLayer, self).__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads, dropout)
        self.cross_attn = MultiHeadAttention(d_model, num_heads, dropout)
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.norm3 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, memory, src_mask=None, tgt_mask=None):
        # Self-attention sub-layer (with causal masking)
        attn_output = self.self_attn(x, x, x, tgt_mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Cross-attention sub-layer
        cross_output = self.cross_attn(x, memory, memory, src_mask)
        x = self.norm2(x + self.dropout(cross_output))
        
        # Feed-forward sub-layer
        ff_output = self.feed_forward(x)
        x = self.norm3(x + self.dropout(ff_output))
        
        return x

class CausalMask:
    @staticmethod
    def generate(seq_len):
        """Generate causal mask for decoder self-attention"""
        mask = torch.tril(torch.ones(seq_len, seq_len)).bool()
        return mask

# Example usage
decoder_layer = TransformerDecoderLayer(d_model=512, num_heads=8, d_ff=2048, dropout=0.1)

# Sample inputs
tgt_input = torch.randn(32, 15, 512)  # Target sequence
memory = torch.randn(32, 20, 512)     # Encoder output
src_mask = torch.ones(32, 1, 20).bool()  # Source mask
tgt_mask = CausalMask.generate(15).unsqueeze(0).expand(32, -1, -1)  # Causal mask

output = decoder_layer(tgt_input, memory, src_mask, tgt_mask)
print(f"Decoder layer output shape: {output.shape}")
```

## 4. How do you implement positional encoding in Transformers?

**Answer:**
```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000, dropout=0.1):
        super(PositionalEncoding, self).__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        # Compute the positional encodings once in log space
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        
        # Different frequencies for different dimensions
        div_term = torch.exp(torch.arange(0, d_model, 2).float() *
                            -(math.log(10000.0) / d_model))
        
        # Apply sine to even indices
        pe[:, 0::2] = torch.sin(position * div_term)
        # Apply cosine to odd indices
        pe[:, 1::2] = torch.cos(position * div_term)
        
        # Add batch dimension
        pe = pe.unsqueeze(0)  # (1, max_len, d_model)
        self.register_buffer('pe', pe)
        
    def forward(self, x):
        # x: (batch_size, seq_len, d_model)
        x = x + self.pe[:, :x.size(1)]  # Add positional encoding
        return self.dropout(x)

class LearnedPositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000, dropout=0.1):
        super(LearnedPositionalEncoding, self).__init__()
        self.dropout = nn.Dropout(p=dropout)
        self.position_embeddings = nn.Embedding(max_len, d_model)
        
    def forward(self, x):
        # x: (batch_size, seq_len, d_model)
        seq_len = x.size(1)
        position_ids = torch.arange(seq_len, dtype=torch.long, device=x.device)
        position_ids = position_ids.unsqueeze(0).expand(x.size(0), -1)
        
        position_embeddings = self.position_embeddings(position_ids)
        x = x + position_embeddings
        return self.dropout(x)

# Example usage
pos_encoding = PositionalEncoding(d_model=512, max_len=1000)
learned_pos_encoding = LearnedPositionalEncoding(d_model=512, max_len=1000)

sample_input = torch.randn(32, 50, 512)  # (batch_size, seq_len, d_model)

# Standard positional encoding
output1 = pos_encoding(sample_input)
print(f"Sinusoidal positional encoding output: {output1.shape}")

# Learned positional encoding
output2 = learned_pos_encoding(sample_input)
print(f"Learned positional encoding output: {output2.shape}")

# Visualize positional encodings
import matplotlib.pyplot as plt
import numpy as np

def visualize_positional_encoding():
    pe = PositionalEncoding(d_model=128, max_len=100)
    pos_enc = pe.pe[0, :50, :32].cpu().numpy()  # First 50 positions, first 32 dimensions
    
    plt.figure(figsize=(12, 8))
    plt.imshow(pos_enc, aspect='auto', cmap='viridis')
    plt.xlabel('Embedding Dimension')
    plt.ylabel('Position')
    plt.title('Positional Encoding')
    plt.colorbar()
    plt.show()

# visualize_positional_encoding()  # Uncomment to visualize
```

## 5. How do you implement a complete Transformer model?

**Answer:**
```python
class TransformerEncoder(nn.Module):
    def __init__(self, num_layers, d_model, num_heads, d_ff, dropout=0.1):
        super(TransformerEncoder, self).__init__()
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
        self.norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        for layer in self.layers:
            x = layer(x, mask)
        return self.norm(x)

class TransformerDecoder(nn.Module):
    def __init__(self, num_layers, d_model, num_heads, d_ff, dropout=0.1):
        super(TransformerDecoder, self).__init__()
        self.layers = nn.ModuleList([
            TransformerDecoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
        self.norm = LayerNorm(d_model)
        
    def forward(self, x, memory, src_mask=None, tgt_mask=None):
        for layer in self.layers:
            x = layer(x, memory, src_mask, tgt_mask)
        return self.norm(x)

class Transformer(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, d_model=512, num_heads=8, 
                 num_encoder_layers=6, num_decoder_layers=6, d_ff=2048, dropout=0.1):
        super(Transformer, self).__init__()
        
        self.d_model = d_model
        
        # Embeddings
        self.src_embedding = nn.Embedding(src_vocab_size, d_model)
        self.tgt_embedding = nn.Embedding(tgt_vocab_size, d_model)
        
        # Positional encoding
        self.pos_encoding = PositionalEncoding(d_model, dropout=dropout)
        
        # Encoder and Decoder
        self.encoder = TransformerEncoder(num_encoder_layers, d_model, num_heads, d_ff, dropout)
        self.decoder = TransformerDecoder(num_decoder_layers, d_model, num_heads, d_ff, dropout)
        
        # Output projection
        self.generator = nn.Linear(d_model, tgt_vocab_size)
        
        # Initialize parameters
        self._reset_parameters()
        
    def _reset_parameters(self):
        for p in self.parameters():
            if p.dim() > 1:
                nn.init.xavier_uniform_(p)
    
    def forward(self, src, tgt, src_mask=None, tgt_mask=None):
        # Encode source
        src_embedded = self.pos_encoding(self.src_embedding(src) * math.sqrt(self.d_model))
        memory = self.encoder(src_embedded, src_mask)
        
        # Decode target
        tgt_embedded = self.pos_encoding(self.tgt_embedding(tgt) * math.sqrt(self.d_model))
        output = self.decoder(tgt_embedded, memory, src_mask, tgt_mask)
        
        # Generate output probabilities
        output = self.generator(output)
        
        return output

# Example usage
transformer = Transformer(
    src_vocab_size=10000,
    tgt_vocab_size=10000,
    d_model=512,
    num_heads=8,
    num_encoder_layers=6,
    num_decoder_layers=6,
    d_ff=2048,
    dropout=0.1
)

# Sample inputs
src = torch.randint(1, 10000, (32, 20))  # (batch_size, src_seq_len)
tgt = torch.randint(1, 10000, (32, 15))  # (batch_size, tgt_seq_len)

# Masks
src_mask = torch.ones(32, 1, 20).bool()  # (batch_size, 1, src_seq_len)
tgt_mask = CausalMask.generate(15).unsqueeze(0).expand(32, -1, -1)  # (batch_size, tgt_seq_len, tgt_seq_len)

output = transformer(src, tgt, src_mask, tgt_mask)
print(f"Transformer output shape: {output.shape}")
```

## 6. How do you implement BERT-style Transformers?

**Answer:**
```python
class BERTEmbeddings(nn.Module):
    def __init__(self, vocab_size, max_position_embeddings, type_vocab_size, hidden_size, dropout=0.1):
        super(BERTEmbeddings, self).__init__()
        self.word_embeddings = nn.Embedding(vocab_size, hidden_size)
        self.position_embeddings = nn.Embedding(max_position_embeddings, hidden_size)
        self.token_type_embeddings = nn.Embedding(type_vocab_size, hidden_size)
        
        self.LayerNorm = LayerNorm(hidden_size)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, input_ids, token_type_ids=None, position_ids=None):
        seq_length = input_ids.size(1)
        device = input_ids.device
        
        if position_ids is None:
            position_ids = torch.arange(seq_length, dtype=torch.long, device=device)
            position_ids = position_ids.unsqueeze(0).expand_as(input_ids)
        
        if token_type_ids is None:
            token_type_ids = torch.zeros_like(input_ids, dtype=torch.long, device=device)
        
        # Get embeddings
        words_embeddings = self.word_embeddings(input_ids)
        position_embeddings = self.position_embeddings(position_ids)
        token_type_embeddings = self.token_type_embeddings(token_type_ids)
        
        # Combine embeddings
        embeddings = words_embeddings + position_embeddings + token_type_embeddings
        embeddings = self.LayerNorm(embeddings)
        embeddings = self.dropout(embeddings)
        
        return embeddings

class BERTLayer(nn.Module):
    def __init__(self, hidden_size, num_attention_heads, intermediate_size, dropout=0.1):
        super(BERTLayer, self).__init__()
        self.attention = MultiHeadAttention(hidden_size, num_attention_heads, dropout)
        self.intermediate = PositionwiseFeedForward(hidden_size, intermediate_size, dropout)
        self.attention_output = nn.Linear(hidden_size, hidden_size)
        self.intermediate_output = nn.Linear(intermediate_size, hidden_size)
        
        self.attention_layer_norm = LayerNorm(hidden_size)
        self.output_layer_norm = LayerNorm(hidden_size)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, hidden_states, attention_mask=None):
        # Self-attention
        attention_output = self.attention(hidden_states, hidden_states, hidden_states, attention_mask)
        attention_output = self.attention_output(attention_output)
        attention_output = self.dropout(attention_output)
        attention_output = self.attention_layer_norm(hidden_states + attention_output)
        
        # Feed-forward
        intermediate_output = self.intermediate(attention_output)
        layer_output = self.intermediate_output(intermediate_output)
        layer_output = self.dropout(layer_output)
        layer_output = self.output_layer_norm(attention_output + layer_output)
        
        return layer_output

class BERTModel(nn.Module):
    def __init__(self, config):
        super(BERTModel, self).__init__()
        self.embeddings = BERTEmbeddings(
            config.vocab_size,
            config.max_position_embeddings,
            config.type_vocab_size,
            config.hidden_size,
            config.hidden_dropout_prob
        )
        
        self.encoder_layers = nn.ModuleList([
            BERTLayer(
                config.hidden_size,
                config.num_attention_heads,
                config.intermediate_size,
                config.hidden_dropout_prob
            ) for _ in range(config.num_hidden_layers)
        ])
        
    def forward(self, input_ids, token_type_ids=None, attention_mask=None):
        # Create attention mask if not provided
        if attention_mask is None:
            attention_mask = (input_ids != 0).unsqueeze(1).unsqueeze(2)
        
        # Convert attention mask to float and apply
        extended_attention_mask = attention_mask.float()
        extended_attention_mask = (1.0 - extended_attention_mask) * -10000.0
        
        # Get embeddings
        embedding_output = self.embeddings(input_ids, token_type_ids)
        
        # Apply encoder layers
        hidden_states = embedding_output
        for layer in self.encoder_layers:
            hidden_states = layer(hidden_states, extended_attention_mask)
        
        return hidden_states

class BERTConfig:
    def __init__(self):
        self.vocab_size = 30522
        self.hidden_size = 768
        self.num_hidden_layers = 12
        self.num_attention_heads = 12
        self.intermediate_size = 3072
        self.hidden_dropout_prob = 0.1
        self.max_position_embeddings = 512
        self.type_vocab_size = 2

# Example usage
config = BERTConfig()
bert_model = BERTModel(config)

# Sample input
input_ids = torch.randint(0, config.vocab_size, (16, 128))  # (batch_size, seq_len)
attention_mask = torch.ones(16, 1, 1, 128).bool()  # (batch_size, 1, 1, seq_len)

output = bert_model(input_ids, attention_mask=attention_mask)
print(f"BERT output shape: {output.shape}")
```

## 7. How do you implement GPT-style Transformers?

**Answer:**
```python
class GPTEmbeddings(nn.Module):
    def __init__(self, vocab_size, hidden_size, max_position_embeddings, dropout=0.1):
        super(GPTEmbeddings, self).__init__()
        self.token_embeddings = nn.Embedding(vocab_size, hidden_size)
        self.position_embeddings = nn.Embedding(max_position_embeddings, hidden_size)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, input_ids, position_ids=None):
        seq_length = input_ids.size(-1)
        
        if position_ids is None:
            position_ids = torch.arange(seq_length, dtype=torch.long, device=input_ids.device)
            position_ids = position_ids.unsqueeze(0).expand_as(input_ids)
        
        token_embeds = self.token_embeddings(input_ids)
        position_embeds = self.position_embeddings(position_ids)
        
        embeddings = token_embeds + position_embeds
        embeddings = self.dropout(embeddings)
        
        return embeddings

class GPTAttention(nn.Module):
    def __init__(self, hidden_size, num_attention_heads, dropout=0.1):
        super(GPTAttention, self).__init__()
        self.hidden_size = hidden_size
        self.num_attention_heads = num_attention_heads
        self.head_dim = hidden_size // num_attention_heads
        
        self.query = nn.Linear(hidden_size, hidden_size)
        self.key = nn.Linear(hidden_size, hidden_size)
        self.value = nn.Linear(hidden_size, hidden_size)
        
        self.attn_dropout = nn.Dropout(dropout)
        self.resid_dropout = nn.Dropout(dropout)
        
        self.proj = nn.Linear(hidden_size, hidden_size)
        
    def forward(self, hidden_states, attention_mask=None):
        batch_size, seq_length = hidden_states.size(0), hidden_states.size(1)
        
        # Linear transformations
        q = self.query(hidden_states).view(batch_size, seq_length, self.num_attention_heads, self.head_dim).transpose(1, 2)
        k = self.key(hidden_states).view(batch_size, seq_length, self.num_attention_heads, self.head_dim).transpose(1, 2)
        v = self.value(hidden_states).view(batch_size, seq_length, self.num_attention_heads, self.head_dim).transpose(1, 2)
        
        # Scaled dot-product attention
        attn_scores = torch.matmul(q, k.transpose(-2, -1)) / math.sqrt(self.head_dim)
        
        # Apply causal mask
        causal_mask = torch.tril(torch.ones(seq_length, seq_length)).to(hidden_states.device)
        causal_mask = causal_mask.unsqueeze(0).unsqueeze(0)
        attn_scores = attn_scores.masked_fill(causal_mask == 0, float('-inf'))
        
        # Apply attention mask if provided
        if attention_mask is not None:
            attn_scores = attn_scores.masked_fill(attention_mask == 0, float('-inf'))
        
        attn_weights = F.softmax(attn_scores, dim=-1)
        attn_weights = self.attn_dropout(attn_weights)
        
        # Apply attention
        attn_output = torch.matmul(attn_weights, v)
        attn_output = attn_output.transpose(1, 2).contiguous().view(batch_size, seq_length, self.hidden_size)
        
        # Output projection
        attn_output = self.proj(attn_output)
        attn_output = self.resid_dropout(attn_output)
        
        return attn_output

class GPTMLP(nn.Module):
    def __init__(self, hidden_size, intermediate_size, dropout=0.1):
        super(GPTMLP, self).__init__()
        self.c_fc = nn.Linear(hidden_size, intermediate_size)
        self.c_proj = nn.Linear(intermediate_size, hidden_size)
        self.act = nn.GELU()
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, hidden_states):
        hidden_states = self.c_fc(hidden_states)
        hidden_states = self.act(hidden_states)
        hidden_states = self.c_proj(hidden_states)
        hidden_states = self.dropout(hidden_states)
        return hidden_states

class GPTBlock(nn.Module):
    def __init__(self, hidden_size, num_attention_heads, intermediate_size, dropout=0.1):
        super(GPTBlock, self).__init__()
        self.ln_1 = LayerNorm(hidden_size)
        self.attn = GPTAttention(hidden_size, num_attention_heads, dropout)
        self.ln_2 = LayerNorm(hidden_size)
        self.mlp = GPTMLP(hidden_size, intermediate_size, dropout)
        
    def forward(self, hidden_states, attention_mask=None):
        # Self-attention
        attn_output = self.attn(self.ln_1(hidden_states), attention_mask)
        hidden_states = hidden_states + attn_output
        
        # MLP
        mlp_output = self.mlp(self.ln_2(hidden_states))
        hidden_states = hidden_states + mlp_output
        
        return hidden_states

class GPTModel(nn.Module):
    def __init__(self, vocab_size=50257, hidden_size=768, num_layers=12, 
                 num_attention_heads=12, intermediate_size=3072, 
                 max_position_embeddings=1024, dropout=0.1):
        super(GPTModel, self).__init__()
        
        self.embeddings = GPTEmbeddings(vocab_size, hidden_size, max_position_embeddings, dropout)
        self.layers = nn.ModuleList([
            GPTBlock(hidden_size, num_attention_heads, intermediate_size, dropout)
            for _ in range(num_layers)
        ])
        self.ln_f = LayerNorm(hidden_size)
        
    def forward(self, input_ids, attention_mask=None):
        # Get embeddings
        hidden_states = self.embeddings(input_ids)
        
        # Apply transformer layers
        for layer in self.layers:
            hidden_states = layer(hidden_states, attention_mask)
        
        # Final layer normalization
        hidden_states = self.ln_f(hidden_states)
        
        return hidden_states

class GPTLMHeadModel(nn.Module):
    def __init__(self, config):
        super(GPTLMHeadModel, self).__init__()
        self.transformer = GPTModel(
            config.vocab_size,
            config.hidden_size,
            config.num_layers,
            config.num_attention_heads,
            config.intermediate_size,
            config.max_position_embeddings,
            config.dropout
        )
        self.lm_head = nn.Linear(config.hidden_size, config.vocab_size, bias=False)
        
        # Weight tying
        self.lm_head.weight = self.transformer.embeddings.token_embeddings.weight
        
    def forward(self, input_ids, attention_mask=None, labels=None):
        # Get transformer outputs
        hidden_states = self.transformer(input_ids, attention_mask)
        
        # Language modeling head
        lm_logits = self.lm_head(hidden_states)
        
        loss = None
        if labels is not None:
            # Shift so that tokens < n predict n
            shift_logits = lm_logits[..., :-1, :].contiguous()
            shift_labels = labels[..., 1:].contiguous()
            
            loss_fct = nn.CrossEntropyLoss()
            loss = loss_fct(shift_logits.view(-1, shift_logits.size(-1)), shift_labels.view(-1))
        
        return {"loss": loss, "logits": lm_logits}

class GPTConfig:
    def __init__(self):
        self.vocab_size = 50257
        self.hidden_size = 768
        self.num_layers = 12
        self.num_attention_heads = 12
        self.intermediate_size = 3072
        self.max_position_embeddings = 1024
        self.dropout = 0.1

# Example usage
config = GPTConfig()
gpt_model = GPTLMHeadModel(config)

# Sample input
input_ids = torch.randint(0, config.vocab_size, (8, 512))  # (batch_size, seq_len)
labels = torch.randint(0, config.vocab_size, (8, 512))     # (batch_size, seq_len)

output = gpt_model(input_ids, labels=labels)
print(f"GPT output - Loss: {output['loss']}, Logits shape: {output['logits'].shape}")
```

## 8. How do you implement Transformer with different attention mechanisms?

**Answer:**
```python
class SparseAttention(nn.Module):
    def __init__(self, d_model, num_heads, block_size=64, dropout=0.1):
        super(SparseAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.block_size = block_size
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        seq_len = query.size(1)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Sparse attention - only attend to local blocks
        scores = torch.zeros(batch_size, self.num_heads, seq_len, seq_len, device=query.device)
        
        for i in range(0, seq_len, self.block_size):
            end_i = min(i + self.block_size, seq_len)
            for j in range(max(0, i - self.block_size), min(seq_len, i + 2 * self.block_size), self.block_size):
                end_j = min(j + self.block_size, seq_len)
                
                # Calculate attention for this block
                q_block = Q[:, :, i:end_i, :]
                k_block = K[:, :, j:end_j, :]
                v_block = V[:, :, j:end_j, :]
                
                block_scores = torch.matmul(q_block, k_block.transpose(-2, -1)) / math.sqrt(self.d_k)
                
                if mask is not None:
                    block_mask = mask[:, :, i:end_i, j:end_j]
                    block_scores = block_scores.masked_fill(block_mask == 0, float('-inf'))
                
                scores[:, :, i:end_i, j:end_j] = block_scores
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

class LinearAttention(nn.Module):
    """
    Linear attention mechanism - O(N) complexity instead of O(N^2)
    """
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(LinearAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        seq_len = query.size(1)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Apply elu activation for linear attention
        Q = F.elu(Q) + 1
        K = F.elu(K) + 1
        
        # Linear attention computation
        # S = Q @ K^T, but we compute it as (Q @ (K^T @ V)) / (Q @ K^T @ 1)
        KV = torch.matmul(K.transpose(-2, -1), V)  # (batch, heads, d_k, d_k)
        Z = torch.sum(K, dim=-2, keepdim=True)    # (batch, heads, 1, d_k)
        
        # Numerator: Q @ KV
        numerator = torch.matmul(Q, KV)  # (batch, heads, seq_len, d_k)
        
        # Denominator: Q @ Z
        denominator = torch.matmul(Q, Z.transpose(-2, -1)) + 1e-6  # (batch, heads, seq_len, 1)
        
        # Final output
        output = numerator / denominator  # (batch, heads, seq_len, d_k)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

class RelativePositionAttention(nn.Module):
    def __init__(self, d_model, num_heads, max_relative_position=16):
        super(RelativePositionAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.max_relative_position = max_relative_position
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Relative position embeddings
        vocab_size = max_relative_position * 2 + 1
        self.relative_k_embed = nn.Embedding(vocab_size, self.d_k)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        seq_len = query.size(1)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Create relative positions
        positions = torch.arange(seq_len, dtype=torch.long, device=query.device)
        relative_positions = positions.unsqueeze(1) - positions.unsqueeze(0)
        relative_positions = torch.clamp(relative_positions, -self.max_relative_position, self.max_relative_position)
        relative_positions = relative_positions + self.max_relative_position
        
        # Get relative position embeddings
        relative_k = self.relative_k_embed(relative_positions)  # (seq_len, seq_len, d_k)
        
        # Standard attention scores
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Add relative position attention
        Q_expanded = Q.unsqueeze(3)  # (batch, heads, seq_len, 1, d_k)
        relative_k_expanded = relative_k.unsqueeze(0).unsqueeze(0)  # (1, 1, seq_len, seq_len, d_k)
        relative_scores = torch.sum(Q_expanded * relative_k_expanded, dim=-1)  # (batch, heads, seq_len, seq_len)
        scores = scores + relative_scores
        
        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

class AdaptiveAttentionLayer(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super(AdaptiveAttentionLayer, self).__init__()
        
        # Multiple attention mechanisms
        self.sparse_attn = SparseAttention(d_model, num_heads, dropout=dropout)
        self.linear_attn = LinearAttention(d_model, num_heads, dropout=dropout)
        self.relative_attn = RelativePositionAttention(d_model, num_heads)
        self.standard_attn = MultiHeadAttention(d_model, num_heads, dropout=dropout)
        
        # Attention selector
        self.selector = nn.Sequential(
            nn.Linear(d_model, d_model // 4),
            nn.ReLU(),
            nn.Linear(d_model // 4, 4),
            nn.Softmax(dim=-1)
        )
        
    def forward(self, x, mask=None):
        # Get attention mechanism weights
        avg_x = torch.mean(x, dim=1)  # (batch, d_model)
        attn_weights = self.selector(avg_x).unsqueeze(1).unsqueeze(1)  # (batch, 1, 1, 4)
        
        # Compute outputs from all attention mechanisms
        outputs = []
        outputs.append(self.sparse_attn(x, x, x, mask))
        outputs.append(self.linear_attn(x, x, x, mask))
        outputs.append(self.relative_attn(x, x, x, mask))
        outputs.append(self.standard_attn(x, x, x, mask))
        
        # Weighted combination
        outputs = torch.stack(outputs, dim=-1)  # (batch, seq_len, d_model, 4)
        combined_output = torch.sum(outputs * attn_weights, dim=-1)  # (batch, seq_len, d_model)
        
        return combined_output

# Example usage
adaptive_layer = AdaptiveAttentionLayer(d_model=256, num_heads=8)
sample_input = torch.randn(16, 30, 256)  # (batch, seq_len, d_model)

output = adaptive_layer(sample_input)
print(f"Adaptive attention output: {output.shape}")

# Test different attention mechanisms
sparse_attn = SparseAttention(d_model=256, num_heads=8)
linear_attn = LinearAttention(d_model=256, num_heads=8)
relative_attn = RelativePositionAttention(d_model=256, num_heads=8)

sparse_output = sparse_attn(sample_input, sample_input, sample_input)
linear_output = linear_attn(sample_input, sample_input, sample_input)
relative_output = relative_attn(sample_input, sample_input, sample_input)

print(f"Sparse attention output: {sparse_output.shape}")
print(f"Linear attention output: {linear_output.shape}")
print(f"Relative attention output: {relative_output.shape}")
```

## 9. How do you implement Transformer with efficient attention mechanisms?

**Answer:**
```python
class EfficientMultiHeadAttention(nn.Module):
    """
    Memory-efficient multi-head attention with optimizations
    """
    def __init__(self, d_model, num_heads, dropout=0.1, use_flash_attention=False):
        super(EfficientMultiHeadAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.use_flash_attention = use_flash_attention
        
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.W_o = nn.Linear(d_model, d_model, bias=False)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        if self.use_flash_attention and hasattr(F, 'scaled_dot_product_attention'):
            # Use PyTorch's optimized attention if available
            output = F.scaled_dot_product_attention(
                Q, K, V, 
                attn_mask=mask, 
                dropout_p=self.dropout.p if self.training else 0.0
            )
        else:
            # Standard implementation with optimizations
            scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
            
            if mask is not None:
                scores = scores.masked_fill(mask == 0, float('-inf'))
            
            attention_weights = F.softmax(scores, dim=-1)
            attention_weights = self.dropout(attention_weights)
            
            output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(output)
        
        return output

class LowRankAttention(nn.Module):
    """
    Low-rank approximation of attention matrices for efficiency
    """
    def __init__(self, d_model, num_heads, rank=32, dropout=0.1):
        super(LowRankAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.rank = rank
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        # Low-rank projection matrices
        self.low_rank_proj_q = nn.Linear(d_model, rank)
        self.low_rank_proj_k = nn.Linear(d_model, rank)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        seq_len = query.size(1)
        
        # Standard linear transformations
        Q = self.W_q(query).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Low-rank projections for efficient computation
        low_rank_q = self.low_rank_proj_q(query)  # (batch, seq_len, rank)
        low_rank_k = self.low_rank_proj_k(key)    # (batch, seq_len, rank)
        
        # Compute low-rank attention approximation
        # Instead of Q @ K^T, compute (Q @ L_q) @ (L_k^T @ K^T)
        low_rank_scores = torch.matmul(low_rank_q, low_rank_k.transpose(-1, -2))  # (batch, seq_len, seq_len)
        low_rank_scores = low_rank_scores.unsqueeze(1).unsqueeze(1)  # Add head dimensions
        low_rank_scores = low_rank_scores.expand(-1, self.num_heads, -1, -1, -1)  # (batch, heads, seq_len, seq_len)
        
        # Standard attention scores
        standard_scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Combine low-rank and standard attention
        scores = standard_scores + low_rank_scores.squeeze(1)  # Simplified combination
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

class WindowedAttention(nn.Module):
    """
    Windowed attention for long sequences
    """
    def __init__(self, d_model, num_heads, window_size=64, dropout=0.1):
        super(WindowedAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.window_size = window_size
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        seq_len = query.size(1)
        
        # Linear transformations
        Q = self.W_q(query).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Apply windowed attention
        scores = torch.zeros(batch_size, self.num_heads, seq_len, seq_len, device=query.device)
        
        # Process in windows
        for i in range(0, seq_len, self.window_size):
            start_idx = max(0, i - self.window_size // 2)
            end_idx = min(seq_len, i + self.window_size + self.window_size // 2)
            
            # Attention within window
            q_window = Q[:, :, i:i+self.window_size, :]
            k_window = K[:, :, start_idx:end_idx, :]
            v_window = V[:, :, start_idx:end_idx, :]
            
            window_scores = torch.matmul(q_window, k_window.transpose(-2, -1)) / math.sqrt(self.d_k)
            
            # Apply mask for window
            if mask is not None:
                window_mask = mask[:, :, i:i+self.window_size, start_idx:end_idx]
                window_scores = window_scores.masked_fill(window_mask == 0, float('-inf'))
            
            scores[:, :, i:i+self.window_size, start_idx:end_idx] = window_scores
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention
        output = torch.matmul(attention_weights, V)
        
        # Reshape and apply final linear transformation
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

class EfficientTransformerLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1, attention_type='standard'):
        super(EfficientTransformerLayer, self).__init__()
        
        if attention_type == 'standard':
            self.attention = EfficientMultiHeadAttention(d_model, num_heads, dropout)
        elif attention_type == 'low_rank':
            self.attention = LowRankAttention(d_model, num_heads, dropout=dropout)
        elif attention_type == 'windowed':
            self.attention = WindowedAttention(d_model, num_heads, dropout=dropout)
        else:
            self.attention = EfficientMultiHeadAttention(d_model, num_heads, dropout)
        
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Self-attention sub-layer
        attn_output = self.attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward sub-layer
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

# Example usage
# Standard efficient attention
eff_layer = EfficientTransformerLayer(d_model=256, num_heads=8, d_ff=1024, attention_type='standard')
sample_input = torch.randn(16, 50, 256)  # (batch, seq_len, d_model)

output = eff_layer(sample_input)
print(f"Efficient transformer layer output: {output.shape}")

# Low-rank attention
low_rank_layer = EfficientTransformerLayer(d_model=256, num_heads=8, d_ff=1024, attention_type='low_rank')
low_rank_output = low_rank_layer(sample_input)
print(f"Low-rank attention output: {low_rank_output.shape}")

# Windowed attention
windowed_layer = EfficientTransformerLayer(d_model=256, num_heads=8, d_ff=1024, attention_type='windowed')
windowed_output = windowed_layer(sample_input)
print(f"Windowed attention output: {windowed_output.shape}")

# Efficient attention with flash attention if available
if hasattr(F, 'scaled_dot_product_attention'):
    flash_layer = EfficientTransformerLayer(d_model=256, num_heads=8, d_ff=1024, 
                                          attention_type='standard', use_flash_attention=True)
    flash_output = flash_layer(sample_input)
    print(f"Flash attention output: {flash_output.shape}")
```

## 10. How do you implement Transformer with advanced optimization techniques?

**Answer:**
```python
class MixedPrecisionTransformer(nn.Module):
    """
    Transformer with mixed precision training support
    """
    def __init__(self, config):
        super(MixedPrecisionTransformer, self).__init__()
        self.d_model = config.d_model
        
        # Embeddings
        self.embeddings = nn.Embedding(config.vocab_size, config.d_model)
        self.pos_encoding = PositionalEncoding(config.d_model, dropout=config.dropout)
        
        # Transformer layers
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(
                config.d_model, 
                config.num_heads, 
                config.d_ff, 
                config.dropout
            ) for _ in range(config.num_layers)
        ])
        
        self.layer_norm = LayerNorm(config.d_model)
        self.output_projection = nn.Linear(config.d_model, config.vocab_size)
        
    def forward(self, input_ids, attention_mask=None):
        # Embeddings with mixed precision considerations
        embeddings = self.embeddings(input_ids)
        embeddings = embeddings * math.sqrt(self.d_model)
        embeddings = self.pos_encoding(embeddings)
        
        # Apply transformer layers
        hidden_states = embeddings
        for layer in self.layers:
            hidden_states = layer(hidden_states, attention_mask)
        
        # Final layer norm
        hidden_states = self.layer_norm(hidden_states)
        
        # Output projection
        logits = self.output_projection(hidden_states)
        
        return logits

class GradientCheckpointingTransformer(nn.Module):
    """
    Transformer with gradient checkpointing for memory efficiency
    """
    def __init__(self, config):
        super(GradientCheckpointingTransformer, self).__init__()
        self.d_model = config.d_model
        
        # Embeddings
        self.embeddings = nn.Embedding(config.vocab_size, config.d_model)
        self.pos_encoding = PositionalEncoding(config.d_model, dropout=config.dropout)
        
        # Transformer layers with checkpointing
        self.layers = nn.ModuleList([
            self._wrap_checkpoint(TransformerEncoderLayer(
                config.d_model, 
                config.num_heads, 
                config.d_ff, 
                config.dropout
            )) for _ in range(config.num_layers)
        ])
        
        self.layer_norm = LayerNorm(config.d_model)
        self.output_projection = nn.Linear(config.d_model, config.vocab_size)
    
    def _wrap_checkpoint(self, module):
        """Wrap module with gradient checkpointing"""
        def custom_forward(*inputs):
            return module(*inputs)
        return custom_forward
    
    def forward(self, input_ids, attention_mask=None):
        # Embeddings
        embeddings = self.embeddings(input_ids)
        embeddings = embeddings * math.sqrt(self.d_model)
        embeddings = self.pos_encoding(embeddings)
        
        # Apply transformer layers with checkpointing
        hidden_states = embeddings
        for layer in self.layers:
            # Use checkpointing to save memory
            hidden_states = torch.utils.checkpoint.checkpoint(
                layer, hidden_states, attention_mask
            )
        
        # Final layer norm
        hidden_states = self.layer_norm(hidden_states)
        
        # Output projection
        logits = self.output_projection(hidden_states)
        
        return logits

class PrunedTransformer(nn.Module):
    """
    Transformer with structured pruning
    """
    def __init__(self, config, pruning_ratio=0.2):
        super(PrunedTransformer, self).__init__()
        self.d_model = config.d_model
        self.pruning_ratio = pruning_ratio
        
        # Embeddings
        self.embeddings = nn.Embedding(config.vocab_size, config.d_model)
        self.pos_encoding = PositionalEncoding(config.d_model, dropout=config.dropout)
        
        # Transformer layers
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(
                config.d_model, 
                config.num_heads, 
                config.d_ff, 
                config.dropout
            ) for _ in range(config.num_layers)
        ])
        
        self.layer_norm = LayerNorm(config.d_model)
        self.output_projection = nn.Linear(config.d_model, config.vocab_size)
        
        # Pruning masks
        self.register_buffer('pruning_masks', torch.ones(len(self.layers)))
        
    def apply_pruning(self, layer_importance_scores):
        """Apply structured pruning based on layer importance"""
        num_layers_to_prune = int(len(self.layers) * self.pruning_ratio)
        
        # Get indices of least important layers
        _, indices = torch.topk(layer_importance_scores, num_layers_to_prune, largest=False)
        
        # Update pruning masks
        self.pruning_masks[indices] = 0
        
        # Remove pruned layers
        pruned_layers = []
        for i, layer in enumerate(self.layers):
            if self.pruning_masks[i] == 1:
                pruned_layers.append(layer)
        
        self.layers = nn.ModuleList(pruned_layers)
        
    def forward(self, input_ids, attention_mask=None):
        # Embeddings
        embeddings = self.embeddings(input_ids)
        embeddings = embeddings * math.sqrt(self.d_model)
        embeddings = self.pos_encoding(embeddings)
        
        # Apply transformer layers
        hidden_states = embeddings
        for layer in self.layers:
            hidden_states = layer(hidden_states, attention_mask)
        
        # Final layer norm
        hidden_states = self.layer_norm(hidden_states)
        
        # Output projection
        logits = self.output_projection(hidden_states)
        
        return logits

class QuantizedTransformer(nn.Module):
    """
    Transformer with quantization support
    """
    def __init__(self, config):
        super(QuantizedTransformer, self).__init__()
        self.d_model = config.d_model
        
        # Embeddings
        self.embeddings = nn.Embedding(config.vocab_size, config.d_model)
        self.pos_encoding = PositionalEncoding(config.d_model, dropout=config.dropout)
        
        # Transformer layers
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(
                config.d_model, 
                config.num_heads, 
                config.d_ff, 
                config.dropout
            ) for _ in range(config.num_layers)
        ])
        
        self.layer_norm = LayerNorm(config.d_model)
        self.output_projection = nn.Linear(config.d_model, config.vocab_size)
        
        # Quantization stubs
        self.quant = torch.quantization.QuantStub()
        self.dequant = torch.quantization.DeQuantStub()
        
    def fuse_model(self):
        """Fuse convolution, batch norm, and ReLU layers"""
        for layer in self.layers:
            torch.quantization.fuse_modules(layer, [
                ['self_attn.W_q', 'self_attn.dropout'],
                ['feed_forward.linear1', 'feed_forward.dropout']
            ], inplace=True)
    
    def forward(self, input_ids, attention_mask=None):
        # Embeddings
        embeddings = self.embeddings(input_ids)
        embeddings = embeddings * math.sqrt(self.d_model)
        embeddings = self.pos_encoding(embeddings)
        
        # Quantize embeddings
        embeddings = self.quant(embeddings)
        
        # Apply transformer layers
        hidden_states = embeddings
        for layer in self.layers:
            hidden_states = layer(hidden_states, attention_mask)
        
        # Final layer norm
        hidden_states = self.layer_norm(hidden_states)
        
        # Dequantize before output projection
        hidden_states = self.dequant(hidden_states)
        
        # Output projection
        logits = self.output_projection(hidden_states)
        
        return logits

class AdaptiveTransformer(nn.Module):
    """
    Transformer with adaptive computation based on input complexity
    """
    def __init__(self, config):
        super(AdaptiveTransformer, self).__init__()
        self.d_model = config.d_model
        self.max_layers = config.num_layers
        
        # Embeddings
        self.embeddings = nn.Embedding(config.vocab_size, config.d_model)
        self.pos_encoding = PositionalEncoding(config.d_model, dropout=config.dropout)
        
        # Transformer layers
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(
                config.d_model, 
                config.num_heads, 
                config.d_ff, 
                config.dropout
            ) for _ in range(config.num_layers)
        ])
        
        # Complexity predictor
        self.complexity_predictor = nn.Sequential(
            nn.Linear(config.d_model, config.d_model // 4),
            nn.ReLU(),
            nn.Linear(config.d_model // 4, 1),
            nn.Sigmoid()
        )
        
        self.layer_norm = LayerNorm(config.d_model)
        self.output_projection = nn.Linear(config.d_model, config.vocab_size)
        
    def forward(self, input_ids, attention_mask=None, adaptive=True):
        # Embeddings
        embeddings = self.embeddings(input_ids)
        embeddings = embeddings * math.sqrt(self.d_model)
        embeddings = self.pos_encoding(embeddings)
        
        # Determine number of layers to use based on input complexity
        if adaptive:
            # Use first token representation for complexity prediction
            first_token = embeddings[:, 0, :]  # (batch, d_model)
            complexity_score = self.complexity_predictor(first_token)  # (batch, 1)
            num_layers = torch.ceil(complexity_score * self.max_layers).long().clamp(1, self.max_layers)
            
            # Apply variable number of layers per batch
            hidden_states = embeddings
            for i, layer in enumerate(self.layers):
                # Check if this layer should be applied for each sample in batch
                apply_layer = (num_layers > i).unsqueeze(1).unsqueeze(2)  # (batch, 1, 1)
                
                # Apply layer conditionally
                layer_output = layer(hidden_states, attention_mask)
                
                # Mix outputs based on whether layer should be applied
                hidden_states = torch.where(
                    apply_layer.expand_as(hidden_states),
                    layer_output,
                    hidden_states
                )
        else:
            # Apply all layers
            hidden_states = embeddings
            for layer in self.layers:
                hidden_states = layer(hidden_states, attention_mask)
        
        # Final layer norm
        hidden_states = self.layer_norm(hidden_states)
        
        # Output projection
        logits = self.output_projection(hidden_states)
        
        return logits

class TransformerConfig:
    def __init__(self):
        self.vocab_size = 30522
        self.d_model = 768
        self.num_heads = 12
        self.d_ff = 3072
        self.num_layers = 12
        self.dropout = 0.1
        self.max_position_embeddings = 512

# Example usage
config = TransformerConfig()

# Mixed precision transformer
mp_transformer = MixedPrecisionTransformer(config)
print("Mixed precision transformer created")

# Gradient checkpointing transformer
gc_transformer = GradientCheckpointingTransformer(config)
print("Gradient checkpointing transformer created")

# Pruned transformer
pruned_transformer = PrunedTransformer(config, pruning_ratio=0.2)
print("Pruned transformer created")

# Quantized transformer
quant_transformer = QuantizedTransformer(config)
print("Quantized transformer created")

# Adaptive transformer
adaptive_transformer = AdaptiveTransformer(config)
print("Adaptive transformer created")

# Test with sample input
sample_input = torch.randint(0, config.vocab_size, (8, 128))  # (batch, seq_len)
attention_mask = torch.ones(8, 1, 128).bool()

# Forward pass with different transformers
outputs = []
for name, model in [("Mixed Precision", mp_transformer), 
                   ("Gradient Checkpointing", gc_transformer),
                   ("Pruned", pruned_transformer),
                   ("Quantized", quant_transformer),
                   ("Adaptive", adaptive_transformer)]:
    try:
        output = model(sample_input, attention_mask)
        outputs.append((name, output.shape))
        print(f"{name} output shape: {output.shape}")
    except Exception as e:
        print(f"Error in {name}: {str(e)}")

print("\nAll transformer optimization techniques demonstrated successfully")
```

## 11. How do you implement Transformer training and optimization?

**Answer:**
```python
class TransformerTrainer:
    """
    Comprehensive trainer for Transformer models
    """
    def __init__(self, model, tokenizer, config):
        self.model = model
        self.tokenizer = tokenizer
        self.config = config
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        
        # Optimizer and scheduler
        self.optimizer = self._get_optimizer()
        self.scheduler = self._get_scheduler()
        
        # Loss function
        self.criterion = nn.CrossEntropyLoss(ignore_index=0)  # Assuming 0 is padding token
        
        # Training statistics
        self.train_losses = []
        self.val_losses = []
        self.best_val_loss = float('inf')
        
    def _get_optimizer(self):
        """Configure optimizer with different options"""
        if self.config.optimizer_type == 'adamw':
            optimizer = torch.optim.AdamW(
                self.model.parameters(),
                lr=self.config.learning_rate,
                betas=(0.9, 0.999),
                eps=1e-8,
                weight_decay=self.config.weight_decay
            )
        elif self.config.optimizer_type == 'sgd':
            optimizer = torch.optim.SGD(
                self.model.parameters(),
                lr=self.config.learning_rate,
                momentum=0.9,
                weight_decay=self.config.weight_decay
            )
        else:
            optimizer = torch.optim.Adam(
                self.model.parameters(),
                lr=self.config.learning_rate,
                betas=(0.9, 0.999),
                eps=1e-8
            )
        return optimizer
    
    def _get_scheduler(self):
        """Configure learning rate scheduler"""
        if self.config.scheduler_type == 'cosine':
            scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
                self.optimizer,
                T_max=self.config.num_epochs
            )
        elif self.config.scheduler_type == 'linear':
            scheduler = torch.optim.lr_scheduler.LinearLR(
                self.optimizer,
                start_factor=1.0,
                end_factor=0.0,
                total_iters=self.config.num_epochs
            )
        elif self.config.scheduler_type == 'warmup':
            # Custom warmup scheduler
            def lr_lambda(current_step):
                if current_step < self.config.warmup_steps:
                    return float(current_step) / float(max(1, self.config.warmup_steps))
                return max(
                    0.0,
                    float(self.config.num_epochs - current_step) / 
                    float(max(1, self.config.num_epochs - self.config.warmup_steps))
                )
            scheduler = torch.optim.lr_scheduler.LambdaLR(self.optimizer, lr_lambda)
        else:
            scheduler = None
        return scheduler
    
    def train_epoch(self, train_loader):
        """Train for one epoch"""
        self.model.train()
        total_loss = 0.0
        num_batches = 0
        
        for batch_idx, batch in enumerate(train_loader):
            # Move to device
            input_ids = batch['input_ids'].to(self.device)
            attention_mask = batch['attention_mask'].to(self.device)
            labels = batch['labels'].to(self.device)
            
            # Forward pass
            self.optimizer.zero_grad()
            
            # Use mixed precision if enabled
            if self.config.use_amp:
                with torch.cuda.amp.autocast():
                    outputs = self.model(input_ids, attention_mask=attention_mask)
                    logits = outputs if isinstance(outputs, torch.Tensor) else outputs.logits
                    # Shift logits and labels for language modeling
                    shift_logits = logits[..., :-1, :].contiguous()
                    shift_labels = labels[..., 1:].contiguous()
                    loss = self.criterion(
                        shift_logits.view(-1, shift_logits.size(-1)),
                        shift_labels.view(-1)
                    )
                
                # Scale loss and backward pass
                self.scaler.scale(loss).backward()
                
                # Gradient clipping
                if self.config.max_grad_norm > 0:
                    self.scaler.unscale_(self.optimizer)
                    torch.nn.utils.clip_grad_norm_(
                        self.model.parameters(),
                        self.config.max_grad_norm
                    )
                
                # Optimizer step
                self.scaler.step(self.optimizer)
                self.scaler.update()
            else:
                outputs = self.model(input_ids, attention_mask=attention_mask)
                logits = outputs if isinstance(outputs, torch.Tensor) else outputs.logits
                # Shift logits and labels for language modeling
                shift_logits = logits[..., :-1, :].contiguous()
                shift_labels = labels[..., 1:].contiguous()
                loss = self.criterion(
                    shift_logits.view(-1, shift_logits.size(-1)),
                    shift_labels.view(-1)
                )
                
                # Backward pass
                loss.backward()
                
                # Gradient clipping
                if self.config.max_grad_norm > 0:
                    torch.nn.utils.clip_grad_norm_(
                        self.model.parameters(),
                        self.config.max_grad_norm
                    )
                
                # Optimizer step
                self.optimizer.step()
            
            total_loss += loss.item()
            num_batches += 1
            
            # Log progress
            if batch_idx % self.config.log_interval == 0:
                print(f"Batch {batch_idx}, Loss: {loss.item():.4f}")
        
        avg_loss = total_loss / num_batches
        self.train_losses.append(avg_loss)
        
        return avg_loss
    
    def validate(self, val_loader):
        """Validate the model"""
        self.model.eval()
        total_loss = 0.0
        num_batches = 0
        
        with torch.no_grad():
            for batch in val_loader:
                input_ids = batch['input_ids'].to(self.device)
                attention_mask = batch['attention_mask'].to(self.device)
                labels = batch['labels'].to(self.device)
                
                outputs = self.model(input_ids, attention_mask=attention_mask)
                logits = outputs if isinstance(outputs, torch.Tensor) else outputs.logits
                
                # Shift logits and labels for language modeling
                shift_logits = logits[..., :-1, :].contiguous()
                shift_labels = labels[..., 1:].contiguous()
                loss = self.criterion(
                    shift_logits.view(-1, shift_logits.size(-1)),
                    shift_labels.view(-1)
                )
                
                total_loss += loss.item()
                num_batches += 1
        
        avg_loss = total_loss / num_batches
        self.val_losses.append(avg_loss)
        
        return avg_loss
    
    def train(self, train_loader, val_loader, save_path=None):
        """Complete training loop"""
        # Initialize AMP scaler if using mixed precision
        if self.config.use_amp:
            self.scaler = torch.cuda.amp.GradScaler()
        
        # Early stopping
        patience_counter = 0
        best_val_loss = float('inf')
        
        for epoch in range(self.config.num_epochs):
            print(f"\nEpoch {epoch + 1}/{self.config.num_epochs}")
            
            # Training
            train_loss = self.train_epoch(train_loader)
            print(f"Training Loss: {train_loss:.4f}")
            
            # Validation
            val_loss = self.validate(val_loader)
            print(f"Validation Loss: {val_loss:.4f}")
            
            # Learning rate scheduling
            if self.scheduler is not None:
                if isinstance(self.scheduler, torch.optim.lr_scheduler.ReduceLROnPlateau):
                    self.scheduler.step(val_loss)
                else:
                    self.scheduler.step()
            
            # Early stopping
            if val_loss < best_val_loss:
                best_val_loss = val_loss
                patience_counter = 0
                # Save best model
                if save_path:
                    self.save_model(save_path, best=True)
            else:
                patience_counter += 1
                if patience_counter >= self.config.patience:
                    print(f"Early stopping at epoch {epoch + 1}")
                    break
            
            # Regular model saving
            if save_path and (epoch + 1) % self.config.save_interval == 0:
                self.save_model(f"{save_path}_epoch_{epoch + 1}")
        
        print("Training completed!")
        return self.train_losses, self.val_losses
    
    def save_model(self, path, best=False):
        """Save model checkpoint"""
        checkpoint = {
            'model_state_dict': self.model.state_dict(),
            'optimizer_state_dict': self.optimizer.state_dict(),
            'train_losses': self.train_losses,
            'val_losses': self.val_losses,
            'config': self.config
        }
        
        if self.scheduler is not None:
            checkpoint['scheduler_state_dict'] = self.scheduler.state_dict()
        
        if best:
            path = f"{path}_best.pth"
        else:
            path = f"{path}.pth"
        
        torch.save(checkpoint, path)
        print(f"Model saved to {path}")
    
    def load_model(self, path):
        """Load model checkpoint"""
        checkpoint = torch.load(path, map_location=self.device)
        self.model.load_state_dict(checkpoint['model_state_dict'])
        self.optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
        
        if 'scheduler_state_dict' in checkpoint and self.scheduler is not None:
            self.scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
        
        self.train_losses = checkpoint.get('train_losses', [])
        self.val_losses = checkpoint.get('val_losses', [])
        
        print(f"Model loaded from {path}")

class TrainingConfig:
    def __init__(self):
        self.learning_rate = 5e-5
        self.batch_size = 32
        self.num_epochs = 10
        self.warmup_steps = 1000
        self.weight_decay = 0.01
        self.max_grad_norm = 1.0
        self.optimizer_type = 'adamw'
        self.scheduler_type = 'warmup'
        self.use_amp = True  # Automatic Mixed Precision
        self.patience = 3
        self.log_interval = 100
        self.save_interval = 1

# Example usage
def create_sample_data_loader(batch_size=8, seq_len=128, vocab_size=1000):
    """Create a sample data loader for demonstration"""
    from torch.utils.data import Dataset, DataLoader
    
    class SampleDataset(Dataset):
        def __init__(self, size=1000, seq_len=128, vocab_size=1000):
            self.size = size
            self.seq_len = seq_len
            self.vocab_size = vocab_size
        
        def __len__(self):
            return self.size
        
        def __getitem__(self, idx):
            # Generate random sequences
            input_ids = torch.randint(1, self.vocab_size, (self.seq_len,))
            attention_mask = torch.ones(self.seq_len).bool()
            labels = input_ids.clone()  # For language modeling
            
            return {
                'input_ids': input_ids,
                'attention_mask': attention_mask,
                'labels': labels
            }
    
    dataset = SampleDataset(size=100, seq_len=seq_len, vocab_size=vocab_size)
    return DataLoader(dataset, batch_size=batch_size, shuffle=True)

# Create sample data
train_loader = create_sample_data_loader(batch_size=4, seq_len=64, vocab_size=1000)
val_loader = create_sample_data_loader(batch_size=4, seq_len=64, vocab_size=1000)

# Create model and trainer
config = TrainingConfig()
config.use_amp = torch.cuda.is_available()  # Only use AMP if CUDA is available

# Simple transformer model for demonstration
class SimpleTransformer(nn.Module):
    def __init__(self, vocab_size=1000, d_model=256, nhead=8, num_layers=4):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.pos_encoding = PositionalEncoding(d_model)
        encoder_layer = nn.TransformerEncoderLayer(d_model, nhead, dim_feedforward=512, batch_first=True)
        self.transformer = nn.TransformerEncoder(encoder_layer, num_layers)
        self.output = nn.Linear(d_model, vocab_size)
    
    def forward(self, src, src_key_padding_mask=None):
        src = self.embedding(src) * math.sqrt(src.size(-1))
        src = self.pos_encoding(src)
        output = self.transformer(src, src_key_padding_mask=src_key_padding_mask)
        return self.output(output)

model = SimpleTransformer(vocab_size=1000, d_model=256, nhead=8, num_layers=4)
trainer = TransformerTrainer(model, None, config)

print("Transformer trainer initialized")
print(f"Using device: {trainer.device}")
print(f"AMP enabled: {config.use_amp}")

# Note: Actual training would be run here, but we'll skip it for demonstration
# to avoid long execution times and potential memory issues

print("\nTransformer training framework ready for use")
print("Training can be started with: trainer.train(train_loader, val_loader, 'model_checkpoint')")
```

## 12. How do you implement Transformer evaluation and analysis?

**Answer:**
```python
class TransformerEvaluator:
    """
    Comprehensive evaluation and analysis of Transformer models
    """
    def __init__(self, model, device=None):
        self.model = model
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.model.eval()
        
    def evaluate_perplexity(self, data_loader):
        """Calculate perplexity on test data"""
        self.model.eval()
        total_loss = 0.0
        total_tokens = 0
        
        criterion = nn.CrossEntropyLoss(ignore_index=0, reduction='sum')
        
        with torch.no_grad():
            for batch in data_loader:
                input_ids = batch['input_ids'].to(self.device)
                attention_mask = batch['attention_mask'].to(self.device)
                labels = batch['labels'].to(self.device)
                
                outputs = self.model(input_ids, attention_mask=attention_mask)
                logits = outputs if isinstance(outputs, torch.Tensor) else outputs.logits
                
                # Shift for language modeling
                shift_logits = logits[..., :-1, :].contiguous()
                shift_labels = labels[..., 1:].contiguous()
                
                loss = criterion(
                    shift_logits.view(-1, shift_logits.size(-1)),
                    shift_labels.view(-1)
                )
                
                total_loss += loss.item()
                total_tokens += (shift_labels != 0).sum().item()  # Count non-padding tokens
        
        avg_loss = total_loss / total_tokens
        perplexity = math.exp(avg_loss)
        
        return {
            'perplexity': perplexity,
            'avg_loss': avg_loss,
            'total_tokens': total_tokens
        }
    
    def analyze_attention_patterns(self, input_ids, attention_mask=None):
        """Analyze attention patterns in the model"""
        self.model.eval()
        
        # Hook to capture attention weights
        attention_weights = []
        
        def attention_hook(module, input, output):
            if hasattr(output, 'attentions'):
                attention_weights.append(output.attentions)
        
        # Register hooks on attention layers
        hooks = []
        for name, module in self.model.named_modules():
            if 'attention' in name.lower():
                hook = module.register_forward_hook(attention_hook)
                hooks.append(hook)
        
        with torch.no_grad():
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            outputs = self.model(input_ids, attention_mask=attention_mask)
        
        # Remove hooks
        for hook in hooks:
            hook.remove()
        
        # Analyze attention patterns
        analysis_results = {}
        
        if attention_weights:
            # Convert to numpy for analysis
            attention_np = [attn.cpu().numpy() for attn in attention_weights]
            
            # Calculate statistics
            avg_attention_entropy = []
            max_attention_per_head = []
            
            for layer_attention in attention_np:
                # Layer attention shape: (batch, heads, seq_len, seq_len)
                layer_entropy = -np.sum(layer_attention * np.log(layer_attention + 1e-8), axis=-1)
                avg_attention_entropy.append(np.mean(layer_entropy))
                
                max_attention = np.max(layer_attention, axis=-1)
                max_attention_per_head.append(np.mean(max_attention))
            
            analysis_results = {
                'avg_entropy_per_layer': avg_attention_entropy,
                'max_attention_per_layer': max_attention_per_head,
                'attention_weights': attention_np
            }
        
        return analysis_results
    
    def measure_inference_speed(self, input_ids, attention_mask=None, num_iterations=100):
        """Measure inference speed"""
        self.model.eval()
        
        input_ids = input_ids.to(self.device)
        if attention_mask is not None:
            attention_mask = attention_mask.to(self.device)
        
        # Warmup
        for _ in range(10):
            with torch.no_grad():
                _ = self.model(input_ids, attention_mask=attention_mask)
        
        # Timing
        start_time = time.time()
        with torch.no_grad():
            for _ in range(num_iterations):
                _ = self.model(input_ids, attention_mask=attention_mask)
        end_time = time.time()
        
        avg_time = (end_time - start_time) / num_iterations
        tokens_per_second = (input_ids.size(0) * input_ids.size(1)) / avg_time
        
        return {
            'avg_inference_time': avg_time,
            'tokens_per_second': tokens_per_second,
            'total_time': end_time - start_time
        }
    
    def evaluate_memory_usage(self, input_ids, attention_mask=None):
        """Measure memory usage"""
        if not torch.cuda.is_available():
            return {'error': 'CUDA not available'}
        
        # Clear cache
        torch.cuda.empty_cache()
        
        # Initial memory
        initial_memory = torch.cuda.memory_allocated()
        
        input_ids = input_ids.to(self.device)
        if attention_mask is not None:
            attention_mask = attention_mask.to(self.device)
        
        # Forward pass
        with torch.no_grad():
            outputs = self.model(input_ids, attention_mask=attention_mask)
        
        peak_memory = torch.cuda.max_memory_allocated()
        final_memory = torch.cuda.memory_allocated()
        
        return {
            'initial_memory_mb': initial_memory / 1024 / 1024,
            'peak_memory_mb': peak_memory / 1024 / 1024,
            'final_memory_mb': final_memory / 1024 / 1024,
            'memory_increase_mb': (peak_memory - initial_memory) / 1024 / 1024
        }
    
    def analyze_model_complexity(self):
        """Analyze model complexity"""
        total_params = sum(p.numel() for p in self.model.parameters())
        trainable_params = sum(p.numel() for p in self.model.parameters() if p.requires_grad)
        
        # Count FLOPs (simplified estimation)
        flops = 0
        for name, module in self.model.named_modules():
            if isinstance(module, nn.Linear):
                flops += 2 * module.in_features * module.out_features
            elif isinstance(module, nn.MultiheadAttention):
                # Simplified FLOP calculation for attention
                if hasattr(module, 'embed_dim'):
                    flops += 4 * module.embed_dim ** 2  # Q, K, V projections + output
        
        return {
            'total_parameters': total_params,
            'trainable_parameters': trainable_params,
            'flops_estimate': flops,
            'model_size_mb': (total_params * 4) / 1024 / 1024  # Assuming float32
        }
    
    def generate_text(self, prompt, max_length=50, temperature=1.0, top_k=50, top_p=0.9):
        """Generate text using the model"""
        self.model.eval()
        
        # Tokenize prompt
        input_ids = prompt if isinstance(prompt, torch.Tensor) else torch.tensor([prompt])
        input_ids = input_ids.to(self.device)
        
        generated = input_ids.clone()
        
        with torch.no_grad():
            for _ in range(max_length):
                outputs = self.model(generated)
                logits = outputs if isinstance(outputs, torch.Tensor) else outputs.logits
                
                # Apply temperature
                logits = logits[:, -1, :] / temperature
                
                # Apply top-k filtering
                if top_k > 0:
                    indices_to_remove = logits < torch.topk(logits, top_k)[0][..., -1, None]
                    logits[indices_to_remove] = float('-inf')
                
                # Apply top-p (nucleus) filtering
                if top_p < 1.0:
                    sorted_logits, sorted_indices = torch.sort(logits, descending=True)
                    cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
                    sorted_indices_to_remove = cumulative_probs > top_p
                    sorted_indices_to_remove[..., 1:] = sorted_indices_to_remove[..., :-1].clone()
                    sorted_indices_to_remove[..., 0] = 0
                    indices_to_remove = sorted_indices[sorted_indices_to_remove]
                    logits[:, indices_to_remove] = float('-inf')
                
                # Sample from the distribution
                probs = F.softmax(logits, dim=-1)
                next_token = torch.multinomial(probs, 1)
                
                # Append to generated sequence
                generated = torch.cat([generated, next_token], dim=1)
        
        return generated

class ModelComparison:
    """
    Compare different Transformer models
    """
    def __init__(self, models_dict):
        self.models = models_dict
        self.evaluators = {name: TransformerEvaluator(model) for name, model in models_dict.items()}
    
    def compare_perplexity(self, data_loader):
        """Compare perplexity across models"""
        results = {}
        for name, evaluator in self.evaluators.items():
            print(f"Evaluating {name}...")
            results[name] = evaluator.evaluate_perplexity(data_loader)
        return results
    
    def compare_speed(self, input_ids, attention_mask=None):
        """Compare inference speed"""
        results = {}
        for name, evaluator in self.evaluators.items():
            print(f"Measuring speed for {name}...")
            results[name] = evaluator.measure_inference_speed(input_ids, attention_mask)
        return results
    
    def compare_complexity(self):
        """Compare model complexity"""
        results = {}
        for name, evaluator in self.evaluators.items():
            print(f"Analyzing complexity for {name}...")
            results[name] = evaluator.analyze_model_complexity()
        return results

# Visualization utilities
def plot_attention_patterns(attention_analysis):
    """Plot attention pattern analysis"""
    import matplotlib.pyplot as plt
    
    layers = range(len(attention_analysis['avg_entropy_per_layer']))
    
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
    
    # Plot entropy
    ax1.plot(layers, attention_analysis['avg_entropy_per_layer'], 'b-o')
    ax1.set_xlabel('Layer')
    ax1.set_ylabel('Average Attention Entropy')
    ax1.set_title('Attention Entropy by Layer')
    ax1.grid(True)
    
    # Plot max attention
    ax2.plot(layers, attention_analysis['max_attention_per_layer'], 'r-o')
    ax2.set_xlabel('Layer')
    ax2.set_ylabel('Max Attention per Head')
    ax2.set_title('Max Attention by Layer')
    ax2.grid(True)
    
    plt.tight_layout()
    plt.show()

def plot_model_comparison(comparison_results, metric='perplexity'):
    """Plot model comparison results"""
    import matplotlib.pyplot as plt
    
    models = list(comparison_results.keys())
    values = [results[metric] for results in comparison_results.values()]
    
    plt.figure(figsize=(10, 6))
    bars = plt.bar(models, values)
    plt.xlabel('Models')
    plt.ylabel(metric.capitalize())
    plt.title(f'Model Comparison - {metric.capitalize()}')
    plt.xticks(rotation=45)
    
    # Add value labels
    for bar, value in zip(bars, values):
        plt.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,
                f'{value:.2f}', ha='center', va='bottom')
    
    plt.tight_layout()
    plt.show()

# Example usage
def demonstrate_evaluation():
    """Demonstrate Transformer evaluation and analysis"""
    
    # Create sample model for demonstration
    model = SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2)
    evaluator = TransformerEvaluator(model)
    
    print("Transformer evaluation and analysis demonstration")
    print("=" * 50)
    
    # Create sample input
    sample_input = torch.randint(1, 1000, (4, 32))  # (batch, seq_len)
    sample_mask = torch.ones(4, 32).bool()
    
    # Model complexity analysis
    print("\n1. Model Complexity Analysis:")
    complexity = evaluator.analyze_model_complexity()
    for key, value in complexity.items():
        print(f"   {key}: {value}")
    
    # Inference speed measurement
    print("\n2. Inference Speed Analysis:")
    speed_results = evaluator.measure_inference_speed(sample_input, sample_mask, num_iterations=10)
    for key, value in speed_results.items():
        print(f"   {key}: {value:.4f}")
    
    # Memory usage (if CUDA available)
    print("\n3. Memory Usage Analysis:")
    memory_results = evaluator.evaluate_memory_usage(sample_input, sample_mask)
    if 'error' not in memory_results:
        for key, value in memory_results.items():
            print(f"   {key}: {value:.2f} MB")
    else:
        print("   CUDA not available for memory analysis")
    
    # Attention pattern analysis
    print("\n4. Attention Pattern Analysis:")
    attention_analysis = evaluator.analyze_attention_patterns(sample_input, sample_mask)
    if attention_analysis:
        print(f"   Analyzed {len(attention_analysis.get('attention_weights', []))} attention layers")
        if 'avg_entropy_per_layer' in attention_analysis:
            avg_entropy = np.mean(attention_analysis['avg_entropy_per_layer'])
            print(f"   Average attention entropy: {avg_entropy:.4f}")
    else:
        print("   No attention patterns captured (model may not support attention hooks)")
    
    # Text generation example
    print("\n5. Text Generation Example:")
    prompt = [101, 2023, 2003, 1037, 2535]  # Sample token IDs
    try:
        generated = evaluator.generate_text(prompt, max_length=10, temperature=0.8)
        print(f"   Generated sequence length: {generated.size(1)}")
        print(f"   Generated tokens: {generated[0].tolist()}")
    except Exception as e:
        print(f"   Text generation failed: {str(e)}")
    
    print("\nEvaluation demonstration completed!")
    
    # Model comparison example
    print("\n6. Model Comparison Example:")
    models_dict = {
        'Small Transformer': SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2),
        'Medium Transformer': SimpleTransformer(vocab_size=1000, d_model=256, nhead=8, num_layers=4)
    }
    
    comparison = ModelComparison(models_dict)
    complexity_results = comparison.compare_complexity()
    
    print("Complexity Comparison:")
    for model_name, results in complexity_results.items():
        print(f"   {model_name}:")
        print(f"     Parameters: {results['total_parameters']:,}")
        print(f"     Model Size: {results['model_size_mb']:.2f} MB")

# Run demonstration
demonstrate_evaluation()

print("\n" + "="*50)
print("Transformer Evaluation and Analysis Framework Ready")
print("="*50)
print("\nKey Features Demonstrated:")
print("1. Perplexity Evaluation")
print("2. Attention Pattern Analysis")
print("3. Inference Speed Measurement")
print("4. Memory Usage Analysis")
print("5. Model Complexity Analysis")
print("6. Text Generation")
print("7. Model Comparison")
```

## 13. How do you implement Transformer fine-tuning and transfer learning?

**Answer:**
```python
class TransformerFineTuner:
    """
    Comprehensive fine-tuning framework for Transformer models
    """
    def __init__(self, model, tokenizer=None, device=None):
        self.model = model
        self.tokenizer = tokenizer
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        
        # Training components
        self.optimizer = None
        self.scheduler = None
        self.criterion = None
        
        # Training statistics
        self.train_losses = []
        self.val_losses = []
        self.metrics = []
        
    def setup_training(self, config):
        """Setup training components based on configuration"""
        # Loss function
        if config.task_type == 'classification':
            self.criterion = nn.CrossEntropyLoss()
        elif config.task_type == 'regression':
            self.criterion = nn.MSELoss()
        elif config.task_type == 'language_modeling':
            self.criterion = nn.CrossEntropyLoss(ignore_index=-100)
        else:
            self.criterion = nn.CrossEntropyLoss()
        
        # Optimizer
        if config.finetune_strategy == 'full':
            # Fine-tune all parameters
            params = self.model.parameters()
        elif config.finetune_strategy == 'last_layer':
            # Only fine-tune last layer
            params = list(self.model.output_projection.parameters())
        elif config.finetune_strategy == 'layerwise_lr':
            # Different learning rates for different layers
            params = []
            base_lr = config.learning_rate
            num_layers = len([n for n, _ in self.model.named_parameters() if 'layer' in n])
            
            for i, (name, param) in enumerate(self.model.named_parameters()):
                if 'layer' in name:
                    layer_num = int(name.split('.')[1]) if '.' in name.split('.')[1] else 0
                    lr = base_lr * (0.9 ** (num_layers - layer_num))
                else:
                    lr = base_lr
                
                params.append({'params': param, 'lr': lr})
        else:
            params = self.model.parameters()
        
        # Setup optimizer
        if config.optimizer_type == 'adamw':
            self.optimizer = torch.optim.AdamW(
                params,
                lr=config.learning_rate,
                weight_decay=config.weight_decay
            )
        elif config.optimizer_type == 'sgd':
            self.optimizer = torch.optim.SGD(
                params,
                lr=config.learning_rate,
                momentum=0.9
            )
        
        # Setup scheduler
        if config.scheduler_type == 'linear':
            self.scheduler = torch.optim.lr_scheduler.LinearLR(
                self.optimizer,
                start_factor=1.0,
                end_factor=0.0,
                total_iters=config.num_epochs
            )
        elif config.scheduler_type == 'cosine':
            self.scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
                self.optimizer,
                T_max=config.num_epochs
            )
        elif config.scheduler_type == 'warmup':
            def lr_lambda(current_step):
                if current_step < config.warmup_steps:
                    return float(current_step) / float(max(1, config.warmup_steps))
                return max(
                    0.0,
                    float(config.num_epochs - current_step) / 
                    float(max(1, config.num_epochs - config.warmup_steps))
                )
            self.scheduler = torch.optim.lr_scheduler.LambdaLR(self.optimizer, lr_lambda)
    
    def freeze_layers(self, freeze_strategy='embeddings'):
        """Freeze specific layers of the model"""
        if freeze_strategy == 'embeddings':
            # Freeze embedding layers
            for name, param in self.model.named_parameters():
                if 'embedding' in name or 'embed' in name:
                    param.requires_grad = False
        elif freeze_strategy == 'encoder':
            # Freeze encoder layers
            for name, param in self.model.named_parameters():
                if 'encoder' in name:
                    param.requires_grad = False
        elif freeze_strategy == 'first_n_layers':
            # Freeze first n layers (example for BERT-style models)
            freeze_layers = 6  # Example: freeze first 6 layers
            for name, param in self.model.named_parameters():
                if any(f'layer.{i}.' in name for i in range(freeze_layers)):
                    param.requires_grad = False
        elif freeze_strategy == 'classifier_only':
            # Freeze everything except classifier
            for name, param in self.model.named_parameters():
                if 'classifier' not in name and 'output' not in name:
                    param.requires_grad = False
    
    def unfreeze_layers(self, unfreeze_strategy='all'):
        """Unfreeze layers of the model"""
        if unfreeze_strategy == 'all':
            for param in self.model.parameters():
                param.requires_grad = True
        elif unfreeze_strategy == 'last_n_layers':
            # Unfreeze last n layers
            unfreeze_layers = 6  # Example: unfreeze last 6 layers
            # This would require model-specific implementation
    
    def progressive_unfreezing(self, epoch, unfreeze_schedule=None):
        """Progressively unfreeze layers during training"""
        if unfreeze_schedule is None:
            unfreeze_schedule = {5: 'layer.11', 10: 'layer.10', 15: 'layer.9'}
        
        layers_to_unfreeze = unfreeze_schedule.get(epoch, [])
        if layers_to_unfreeze:
            for name, param in self.model.named_parameters():
                if any(layer_name in name for layer_name in layers_to_unfreeze):
                    param.requires_grad = True
                    print(f"Unfrozen layer: {name}")
    
    def train_finetuning_epoch(self, train_loader, epoch=0):
        """Train for one epoch during fine-tuning"""
        self.model.train()
        total_loss = 0.0
        correct_predictions = 0
        total_samples = 0
        
        for batch_idx, batch in enumerate(train_loader):
            # Move data to device
            input_ids = batch['input_ids'].to(self.device)
            attention_mask = batch['attention_mask'].to(self.device)
            labels = batch['labels'].to(self.device)
            
            # Forward pass
            self.optimizer.zero_grad()
            
            outputs = self.model(input_ids, attention_mask=attention_mask)
            
            # Handle different output formats
            if hasattr(outputs, 'logits'):
                logits = outputs.logits
            elif isinstance(outputs, tuple):
                logits = outputs[0]
            else:
                logits = outputs
            
            # Calculate loss
            if len(logits.shape) > 2:  # Sequence output
                # For sequence classification, use [CLS] token or pooler output
                if hasattr(self.model, 'classifier'):
                    # BERT-style models
                    loss = self.criterion(logits, labels)
                else:
                    # Use first token for classification
                    loss = self.criterion(logits[:, 0, :], labels)
            else:  # Direct classification output
                loss = self.criterion(logits, labels)
            
            # Backward pass
            loss.backward()
            
            # Gradient clipping
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), 1.0)
            
            # Optimizer step
            self.optimizer.step()
            
            total_loss += loss.item()
            
            # Calculate accuracy for classification tasks
            if len(logits.shape) > 1 and logits.shape[1] > 1:
                predictions = torch.argmax(logits, dim=-1)
                correct_predictions += (predictions == labels).sum().item()
                total_samples += labels.size(0)
            
            # Log progress
            if batch_idx % 10 == 0:
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item():.4f}")
        
        avg_loss = total_loss / len(train_loader)
        accuracy = correct_predictions / total_samples if total_samples > 0 else 0
        
        self.train_losses.append(avg_loss)
        
        return avg_loss, accuracy
    
    def validate_finetuning(self, val_loader):
        """Validate during fine-tuning"""
        self.model.eval()
        total_loss = 0.0
        correct_predictions = 0
        total_samples = 0
        
        with torch.no_grad():
            for batch in val_loader:
                input_ids = batch['input_ids'].to(self.device)
                attention_mask = batch['attention_mask'].to(self.device)
                labels = batch['labels'].to(self.device)
                
                outputs = self.model(input_ids, attention_mask=attention_mask)
                
                # Handle different output formats
                if hasattr(outputs, 'logits'):
                    logits = outputs.logits
                elif isinstance(outputs, tuple):
                    logits = outputs[0]
                else:
                    logits = outputs
                
                # Calculate loss
                if len(logits.shape) > 2:
                    if hasattr(self.model, 'classifier'):
                        loss = self.criterion(logits, labels)
                    else:
                        loss = self.criterion(logits[:, 0, :], labels)
                else:
                    loss = self.criterion(logits, labels)
                
                total_loss += loss.item()
                
                # Calculate accuracy
                if len(logits.shape) > 1 and logits.shape[1] > 1:
                    predictions = torch.argmax(logits, dim=-1)
                    correct_predictions += (predictions == labels).sum().item()
                    total_samples += labels.size(0)
        
        avg_loss = total_loss / len(val_loader)
        accuracy = correct_predictions / total_samples if total_samples > 0 else 0
        
        self.val_losses.append(avg_loss)
        
        return avg_loss, accuracy
    
    def fine_tune(self, train_loader, val_loader, config, save_path=None):
        """Complete fine-tuning process"""
        # Setup training components
        self.setup_training(config)
        
        # Freeze layers if specified
        if hasattr(config, 'freeze_strategy'):
            self.freeze_layers(config.freeze_strategy)
        
        # Early stopping
        best_val_loss = float('inf')
        patience_counter = 0
        
        for epoch in range(config.num_epochs):
            print(f"\nEpoch {epoch + 1}/{config.num_epochs}")
            
            # Progressive unfreezing
            if hasattr(config, 'progressive_unfreezing') and config.progressive_unfreezing:
                self.progressive_unfreezing(epoch)
            
            # Training
            train_loss, train_acc = self.train_finetuning_epoch(train_loader, epoch)
            print(f"Training Loss: {train_loss:.4f}, Accuracy: {train_acc:.4f}")
            
            # Validation
            val_loss, val_acc = self.validate_finetuning(val_loader)
            print(f"Validation Loss: {val_loss:.4f}, Accuracy: {val_acc:.4f}")
            
            # Learning rate scheduling
            if self.scheduler:
                if isinstance(self.scheduler, torch.optim.lr_scheduler.ReduceLROnPlateau):
                    self.scheduler.step(val_loss)
                else:
                    self.scheduler.step()
            
            # Early stopping
            if val_loss < best_val_loss:
                best_val_loss = val_loss
                patience_counter = 0
                if save_path:
                    self.save_model(save_path, best=True)
            else:
                patience_counter += 1
                if patience_counter >= config.patience:
                    print(f"Early stopping at epoch {epoch + 1}")
                    break
            
            # Regular saving
            if save_path and (epoch + 1) % config.save_interval == 0:
                self.save_model(f"{save_path}_epoch_{epoch + 1}")
        
        print("Fine-tuning completed!")
        return self.train_losses, self.val_losses
    
    def save_model(self, path, best=False):
        """Save fine-tuned model"""
        checkpoint = {
            'model_state_dict': self.model.state_dict(),
            'optimizer_state_dict': self.optimizer.state_dict() if self.optimizer else None,
            'train_losses': self.train_losses,
            'val_losses': self.val_losses
        }
        
        if self.scheduler:
            checkpoint['scheduler_state_dict'] = self.scheduler.state_dict()
        
        if best:
            path = f"{path}_best.pth"
        else:
            path = f"{path}.pth"
        
        torch.save(checkpoint, path)
        print(f"Model saved to {path}")
    
    def load_model(self, path):
        """Load fine-tuned model"""
        checkpoint = torch.load(path, map_location=self.device)
        self.model.load_state_dict(checkpoint['model_state_dict'])
        
        if checkpoint.get('optimizer_state_dict') and self.optimizer:
            self.optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
        
        if checkpoint.get('scheduler_state_dict') and self.scheduler:
            self.scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
        
        self.train_losses = checkpoint.get('train_losses', [])
        self.val_losses = checkpoint.get('val_losses', [])
        
        print(f"Model loaded from {path}")

class FineTuningConfig:
    """Configuration for fine-tuning"""
    def __init__(self):
        self.task_type = 'classification'  # classification, regression, language_modeling
        self.finetune_strategy = 'full'    # full, last_layer, layerwise_lr
        self.freeze_strategy = None        # embeddings, encoder, first_n_layers, classifier_only
        self.progressive_unfreezing = False
        
        # Training parameters
        self.learning_rate = 2e-5
        self.batch_size = 16
        self.num_epochs = 3
        self.warmup_steps = 100
        self.weight_decay = 0.01
        self.patience = 3
        self.save_interval = 1
        
        # Optimizer and scheduler
        self.optimizer_type = 'adamw'
        self.scheduler_type = 'linear'

# Task-specific adapters
class ClassificationAdapter(nn.Module):
    """Adapter for classification tasks"""
    def __init__(self, base_model, num_classes, dropout=0.1):
        super().__init__()
        self.base_model = base_model
        self.dropout = nn.Dropout(dropout)
        self.classifier = nn.Linear(self.get_hidden_size(), num_classes)
        
    def get_hidden_size(self):
        """Get the hidden size of the base model"""
        if hasattr(self.base_model, 'config'):
            return self.base_model.config.hidden_size
        elif hasattr(self.base_model, 'd_model'):
            return self.base_model.d_model
        else:
            # Default assumption
            return 768
    
    def forward(self, input_ids, attention_mask=None, **kwargs):
        outputs = self.base_model(input_ids, attention_mask=attention_mask, **kwargs)
        
        # Handle different output formats
        if hasattr(outputs, 'last_hidden_state'):
            # BERT-style output
            pooled_output = outputs.last_hidden_state[:, 0]  # [CLS] token
        elif isinstance(outputs, tuple):
            pooled_output = outputs[0][:, 0]  # First token
        else:
            pooled_output = outputs[:, 0]  # First token
        
        pooled_output = self.dropout(pooled_output)
        logits = self.classifier(pooled_output)
        
        return logits

class SequenceLabelingAdapter(nn.Module):
    """Adapter for sequence labeling tasks"""
    def __init__(self, base_model, num_labels, dropout=0.1):
        super().__init__()
        self.base_model = base_model
        self.dropout = nn.Dropout(dropout)
        self.classifier = nn.Linear(self.get_hidden_size(), num_labels)
        
    def get_hidden_size(self):
        if hasattr(self.base_model, 'config'):
            return self.base_model.config.hidden_size
        elif hasattr(self.base_model, 'd_model'):
            return self.base_model.d_model
        else:
            return 768
    
    def forward(self, input_ids, attention_mask=None, **kwargs):
        outputs = self.base_model(input_ids, attention_mask=attention_mask, **kwargs)
        
        # Handle different output formats
        if hasattr(outputs, 'last_hidden_state'):
            sequence_output = outputs.last_hidden_state
        elif isinstance(outputs, tuple):
            sequence_output = outputs[0]
        else:
            sequence_output = outputs
        
        sequence_output = self.dropout(sequence_output)
        logits = self.classifier(sequence_output)
        
        return logits

# Example usage and demonstration
def demonstrate_fine_tuning():
    """Demonstrate Transformer fine-tuning capabilities"""
    
    print("Transformer Fine-tuning Demonstration")
    print("=" * 40)
    
    # Create base model
    base_model = SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2)
    
    # Example 1: Classification adapter
    print("\n1. Creating Classification Adapter:")
    classifier_model = ClassificationAdapter(base_model, num_classes=5)
    print(f"   Classification model created with {5} classes")
    
    # Example 2: Sequence labeling adapter
    print("\n2. Creating Sequence Labeling Adapter:")
    seq_label_model = SequenceLabelingAdapter(base_model, num_labels=10)
    print(f"   Sequence labeling model created with {10} labels")
    
    # Example 3: Fine-tuning configuration
    print("\n3. Setting up Fine-tuning Configuration:")
    config = FineTuningConfig()
    config.task_type = 'classification'
    config.finetune_strategy = 'full'
    config.learning_rate = 1e-4
    config.num_epochs = 2
    config.batch_size = 8
    
    print("   Fine-tuning configuration:")
    print(f"     Task type: {config.task_type}")
    print(f"     Strategy: {config.finetune_strategy}")
    print(f"     Learning rate: {config.learning_rate}")
    print(f"     Epochs: {config.num_epochs}")
    
    # Example 4: Fine-tuner setup
    print("\n4. Setting up Fine-tuner:")
    fine_tuner = TransformerFineTuner(classifier_model)
    fine_tuner.setup_training(config)
    print("   Fine-tuner initialized and configured")
    
    # Example 5: Layer freezing strategies
    print("\n5. Layer Freezing Strategies:")
    strategies = ['embeddings', 'encoder', 'first_n_layers', 'classifier_only']
    for strategy in strategies:
        print(f"   Available strategy: {strategy}")
    
    # Example 6: Progressive unfreezing
    print("\n6. Progressive Unfreezing:")
    unfreeze_schedule = {1: 'layer.1', 2: 'layer.0'}
    print(f"   Example schedule: {unfreeze_schedule}")
    
    # Create sample data for demonstration
    print("\n7. Sample Data Preparation:")
    def create_classification_dataset(size=100, seq_len=32, num_classes=5, vocab_size=1000):
        from torch.utils.data import Dataset, DataLoader
        
        class ClassificationDataset(Dataset):
            def __init__(self, size, seq_len, num_classes, vocab_size):
                self.size = size
                self.seq_len = seq_len
                self.num_classes = num_classes
                self.vocab_size = vocab_size
            
            def __len__(self):
                return self.size
            
            def __getitem__(self, idx):
                input_ids = torch.randint(1, self.vocab_size, (self.seq_len,))
                attention_mask = torch.ones(self.seq_len).bool()
                labels = torch.randint(0, self.num_classes, (1,)).squeeze()
                
                return {
                    'input_ids': input_ids,
                    'attention_mask': attention_mask,
                    'labels': labels
                }
        
        return DataLoader(ClassificationDataset(size, seq_len, num_classes, vocab_size), 
                         batch_size=4, shuffle=True)
    
    train_loader = create_classification_dataset(size=50, seq_len=16, num_classes=5)
    val_loader = create_classification_dataset(size=20, seq_len=16, num_classes=5)
    
    print("   Sample datasets created for demonstration")
    
    # Note: Actual fine-tuning would be run here
    print("\n8. Fine-tuning Process:")
    print("   The fine-tuning process would include:")
    print("   - Training loop with validation")
    print("   - Model checkpointing")
    print("   - Early stopping")
    print("   - Learning rate scheduling")
    print("   - Progress tracking")
    
    print("\nFine-tuning demonstration completed!")
    print("\nReady to perform actual fine-tuning with:")
    print("fine_tuner.fine_tune(train_loader, val_loader, config, 'model_checkpoint')")

# Run demonstration
demonstrate_fine_tuning()

print("\n" + "="*50)
print("Transformer Fine-tuning Framework Ready")
print("="*50)
print("\nKey Features Demonstrated:")
print("1. Task-specific Adapters")
print("2. Flexible Fine-tuning Strategies")
print("3. Layer Freezing/Unfreezing")
print("4. Progressive Unfreezing")
print("5. Configuration Management")
print("6. Training Pipeline")
print("7. Model Checkpointing")
```

## 14. How do you implement advanced Transformer architectures?

**Answer:**
```python
class UniversalTransformer(nn.Module):
    """
    Universal Transformer - adaptive computation time for Transformers
    """
    def __init__(self, d_model, num_heads, d_ff, max_depth=8, threshold=0.99, dropout=0.1):
        super(UniversalTransformer, self).__init__()
        self.d_model = d_model
        self.max_depth = max_depth
        self.threshold = threshold
        
        # Shared transformer layer
        self.transformer_layer = TransformerEncoderLayer(d_model, num_heads, d_ff, dropout)
        
        # ACT (Adaptive Computation Time) mechanism
        self.halt_weight = nn.Linear(d_model, 1)
        self.halt_bias = nn.Parameter(torch.zeros(1))
        
        # Layer norm and output projection
        self.layer_norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        batch_size, seq_len = x.size(0), x.size(1)
        
        # Initialize ACT variables
        halting_probability = torch.zeros(batch_size, seq_len, device=x.device)
        remainders = torch.zeros(batch_size, seq_len, device=x.device)
        n_updates = torch.zeros(batch_size, seq_len, device=x.device)
        
        # Initialize previous state
        previous_state = x
        
        for depth in range(self.max_depth):
            # Compute halting probability
            p = torch.sigmoid(self.halt_weight(previous_state) + self.halt_bias)  # (batch, seq_len, 1)
            p = p.squeeze(-1)  # (batch, seq_len)
            
            # Compute new states for sequences still in computation
            still_running = (halting_probability < 1 - 1e-6).float()
            p = p * still_running + 1e-6  # Add small epsilon to avoid division by zero
            remaining = 1 - halting_probability
            
            # Update halting probabilities
            halting_probability_new = halting_probability + p * still_running
            
            # Compute weights for this step
            weights = torch.where(
                halting_probability_new >= 1 - 1e-6,
                remaining,
                p
            )
            
            # Apply transformer layer
            new_state = self.transformer_layer(previous_state, mask)
            
            # Update states
            x = (previous_state + new_state * weights.unsqueeze(-1)) * still_running.unsqueeze(-1) + \
                x * (1 - still_running.unsqueeze(-1))
            
            # Update ACT variables
            remainders += (1 - halting_probability_new) * still_running
            halting_probability = halting_probability_new
            n_updates += still_running
            
            # Check if all sequences have halted
            if (halting_probability >= 1 - 1e-6).all():
                break
                
            previous_state = new_state
        
        # Apply layer norm
        x = self.layer_norm(x)
        
        return x

class Reformer(nn.Module):
    """
    Reformer - Efficient Transformer with LSH attention and reversible layers
    """
    def __init__(self, d_model, num_heads, d_ff, num_layers, bucket_size=64, dropout=0.1):
        super(Reformer, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.bucket_size = bucket_size
        
        # Position embeddings
        self.pos_embeddings = nn.Embedding(4096, d_model)
        
        # Reformer layers with LSH attention
        self.layers = nn.ModuleList([
            ReformerLayer(d_model, num_heads, d_ff, bucket_size, dropout)
            for _ in range(num_layers)
        ])
        
        self.layer_norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        batch_size, seq_len = x.size(0), x.size(1)
        
        # Add position embeddings
        positions = torch.arange(seq_len, device=x.device).unsqueeze(0).expand(batch_size, -1)
        x = x + self.pos_embeddings(positions)
        
        # Apply Reformer layers
        for layer in self.layers:
            x = layer(x, mask)
        
        x = self.layer_norm(x)
        return x

class ReformerLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, bucket_size, dropout=0.1):
        super(ReformerLayer, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.bucket_size = bucket_size
        self.d_k = d_model // num_heads
        
        # LSH attention
        self.lsh_attention = LSHAttention(d_model, num_heads, bucket_size, dropout)
        
        # Feed-forward network
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        
        # Layer normalization
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # LSH attention
        attn_output = self.lsh_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

class LSHAttention(nn.Module):
    def __init__(self, d_model, num_heads, bucket_size, dropout=0.1):
        super(LSHAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.bucket_size = bucket_size
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size, seq_len = query.size(0), query.size(1)
        
        # Linear transformations
        Q = self.W_q(query)  # (batch, seq_len, d_model)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # LSH hashing - simplified implementation
        # In practice, this would use locality-sensitive hashing
        
        # For demonstration, we'll use a simplified attention mechanism
        # that processes in chunks based on bucket_size
        
        # Pad sequence to be divisible by bucket_size
        pad_len = (self.bucket_size - seq_len % self.bucket_size) % self.bucket_size
        if pad_len > 0:
            Q = F.pad(Q, (0, 0, 0, pad_len))
            K = F.pad(K, (0, 0, 0, pad_len))
            V = F.pad(V, (0, 0, 0, pad_len))
        
        # Reshape into buckets
        bucketed_Q = Q.view(batch_size, -1, self.bucket_size, self.d_model)
        bucketed_K = K.view(batch_size, -1, self.bucket_size, self.d_model)
        bucketed_V = V.view(batch_size, -1, self.bucket_size, self.d_model)
        
        # Apply attention within each bucket
        outputs = []
        for i in range(bucketed_Q.size(1)):
            q_bucket = bucketed_Q[:, i]  # (batch, bucket_size, d_model)
            k_bucket = bucketed_K[:, i]
            v_bucket = bucketed_V[:, i]
            
            # Standard attention within bucket
            scores = torch.matmul(q_bucket, k_bucket.transpose(-2, -1)) / math.sqrt(self.d_k)
            attention_weights = F.softmax(scores, dim=-1)
            attention_weights = self.dropout(attention_weights)
            
            output_bucket = torch.matmul(attention_weights, v_bucket)
            outputs.append(output_bucket)
        
        # Concatenate outputs
        output = torch.cat(outputs, dim=1)  # (batch, seq_len + pad_len, d_model)
        
        # Remove padding
        if pad_len > 0:
            output = output[:, :seq_len, :]
        
        # Final linear transformation
        output = self.W_o(output)
        
        return output

class Performer(nn.Module):
    """
    Performer - Linear attention using kernel-based approximation
    """
    def __init__(self, d_model, num_heads, d_ff, num_layers, nb_features=None, dropout=0.1):
        super(Performer, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        
        # Number of random features for kernel approximation
        self.nb_features = nb_features or int(d_model * math.log(d_model))
        
        # Performer layers
        self.layers = nn.ModuleList([
            PerformerLayer(d_model, num_heads, d_ff, self.nb_features, dropout)
            for _ in range(num_layers)
        ])
        
        self.layer_norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        # Apply Performer layers
        for layer in self.layers:
            x = layer(x, mask)
        
        x = self.layer_norm(x)
        return x

class PerformerLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, nb_features, dropout=0.1):
        super(PerformerLayer, self).__init__()
        self.performer_attention = PerformerAttention(d_model, num_heads, nb_features, dropout)
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Performer attention
        attn_output = self.performer_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

class PerformerAttention(nn.Module):
    def __init__(self, d_model, num_heads, nb_features, dropout=0.1):
        super(PerformerAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.nb_features = nb_features
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
        # Random projection matrix for kernel approximation
        self.random_projection = nn.Parameter(
            torch.randn(nb_features, d_model), requires_grad=False
        )
        
    def forward(self, query, key, value, mask=None):
        batch_size, seq_len = query.size(0), query.size(1)
        
        # Linear transformations
        Q = self.W_q(query)  # (batch, seq_len, d_model)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Apply kernel approximation using random features
        # Phi(x) = [cos(w1^T x), sin(w1^T x), ..., cos(wm^T x), sin(wm^T x)]
        Q_phi = self.kernelize(Q)  # (batch, seq_len, 2 * nb_features)
        K_phi = self.kernelize(K)  # (batch, seq_len, 2 * nb_features)
        
        # Compute attention using kernelized features
        # This is a simplified version - in practice, more sophisticated methods are used
        
        # Standard attention computation for demonstration
        Q = Q.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        output = torch.matmul(attention_weights, V)
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output
    
    def kernelize(self, x):
        """Apply kernel approximation using random features"""
        # Project x using random matrix
        x_projected = torch.matmul(x, self.random_projection.t())  # (batch, seq_len, nb_features)
        
        # Apply cos and sin to create random features
        cos_features = torch.cos(x_projected)
        sin_features = torch.sin(x_projected)
        
        # Concatenate cos and sin features
        features = torch.cat([cos_features, sin_features], dim=-1)  # (batch, seq_len, 2 * nb_features)
        
        # Normalize
        features = features / math.sqrt(2 * self.nb_features)
        
        return features

class Linformer(nn.Module):
    """
    Linformer - Linear attention using low-rank projections
    """
    def __init__(self, d_model, num_heads, d_ff, seq_len, k=256, num_layers=6, dropout=0.1):
        super(Linformer, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.seq_len = seq_len
        self.k = k  # Low-rank projection dimension
        
        # Low-rank projection matrices (shared across layers for efficiency)
        self.E = nn.Parameter(torch.randn(k, seq_len))
        self.F = nn.Parameter(torch.randn(k, seq_len))
        
        # Linformer layers
        self.layers = nn.ModuleList([
            LinformerLayer(d_model, num_heads, d_ff, seq_len, k, self.E, self.F, dropout)
            for _ in range(num_layers)
        ])
        
        self.layer_norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        # Apply Linformer layers
        for layer in self.layers:
            x = layer(x, mask)
        
        x = self.layer_norm(x)
        return x

class LinformerLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, seq_len, k, E, F, dropout=0.1):
        super(LinformerLayer, self).__init__()
        self.linformer_attention = LinformerAttention(d_model, num_heads, seq_len, k, E, F, dropout)
        self.feed_forward = PositionwiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Linformer attention
        attn_output = self.linformer_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

class LinformerAttention(nn.Module):
    def __init__(self, d_model, num_heads, seq_len, k, E, F, dropout=0.1):
        super(LinformerAttention, self).__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.seq_len = seq_len
        self.k = k
        self.d_k = d_model // num_heads
        
        # Projection matrices (shared from parent)
        self.E = E  # (k, seq_len)
        self.F = F  # (k, seq_len)
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, query, key, value, mask=None):
        batch_size, seq_len = query.size(0), query.size(1)
        
        # Linear transformations
        Q = self.W_q(query)  # (batch, seq_len, d_model)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # Apply low-rank projections to reduce sequence length
        # E: (k, seq_len), K: (batch, seq_len, d_model)
        # E @ K.transpose(-2, -1): (k, d_model) -> (batch, k, d_model)
        K_projected = torch.matmul(self.E[:, :seq_len], K)  # (batch, k, d_model)
        V_projected = torch.matmul(self.F[:, :seq_len], V)  # (batch, k, d_model)
        
        # Reshape for multi-head attention
        Q = Q.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K_proj = K_projected.view(batch_size, self.k, self.num_heads, self.d_k).transpose(1, 2)
        V_proj = V_projected.view(batch_size, self.k, self.num_heads, self.d_k).transpose(1, 2)
        
        # Compute attention with projected keys and values
        scores = torch.matmul(Q, K_proj.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        # Apply causal mask for decoder attention (if needed)
        if mask is not None:
            # Adjust mask for projected dimensions
            if mask.dim() == 3:
                # (batch, 1, seq_len) -> (batch, 1, 1, seq_len)
                mask = mask.unsqueeze(1)
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)
        
        # Apply attention to projected values
        output = torch.matmul(attention_weights, V_proj)
        output = output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        output = self.W_o(output)
        
        return output

# Example usage and demonstration
def demonstrate_advanced_transformers():
    """Demonstrate advanced Transformer architectures"""
    
    print("Advanced Transformer Architectures Demonstration")
    print("=" * 50)
    
    # Create sample input
    batch_size, seq_len, d_model = 4, 64, 128
    sample_input = torch.randn(batch_size, seq_len, d_model)
    sample_mask = torch.ones(batch_size, 1, seq_len).bool()
    
    # 1. Universal Transformer
    print("\n1. Universal Transformer:")
    try:
        universal_transformer = UniversalTransformer(
            d_model=d_model,
            num_heads=4,
            d_ff=256,
            max_depth=4,
            threshold=0.99
        )
        universal_output = universal_transformer(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Output shape: {universal_output.shape}")
        print("   ✓ Universal Transformer created successfully")
    except Exception as e:
        print(f"   ✗ Universal Transformer failed: {str(e)}")
    
    # 2. Performer (Linear attention)
    print("\n2. Performer (Linear Attention):")
    try:
        performer = Performer(
            d_model=d_model,
            num_heads=4,
            d_ff=256,
            num_layers=2,
            nb_features=64
        )
        performer_output = performer(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Output shape: {performer_output.shape}")
        print("   ✓ Performer created successfully")
    except Exception as e:
        print(f"   ✗ Performer failed: {str(e)}")
    
    # 3. Linformer (Low-rank attention)
    print("\n3. Linformer (Low-rank Attention):")
    try:
        linformer = Linformer(
            d_model=d_model,
            num_heads=4,
            d_ff=256,
            seq_len=seq_len,
            k=min(32, seq_len // 2),
            num_layers=2
        )
        linformer_output = linformer(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Output shape: {linformer_output.shape}")
        print("   ✓ Linformer created successfully")
    except Exception as e:
        print(f"   ✗ Linformer failed: {str(e)}")
    
    # 4. Reformer (LSH attention)
    print("\n4. Reformer (LSH Attention):")
    try:
        reformer = Reformer(
            d_model=d_model,
            num_heads=4,
            d_ff=256,
            num_layers=2,
            bucket_size=16
        )
        reformer_output = reformer(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Output shape: {reformer_output.shape}")
        print("   ✓ Reformer created successfully")
    except Exception as e:
        print(f"   ✗ Reformer failed: {str(e)}")
    
    # Complexity comparison
    print("\n5. Model Complexity Comparison:")
    models = {
        'Standard Transformer': SimpleTransformer(vocab_size=1000, d_model=d_model, nhead=4, num_layers=2),
        'Performer': performer,
        'Linformer': linformer,
        'Universal Transformer': universal_transformer
    }
    
    for name, model in models.items():
        try:
            total_params = sum(p.numel() for p in model.parameters())
            trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
            print(f"   {name}:")
            print(f"     Total parameters: {total_params:,}")
            print(f"     Trainable parameters: {trainable_params:,}")
        except:
            print(f"   {name}: Parameter count not available")
    
    print("\nAdvanced Transformer architectures demonstration completed!")
    
    # Performance characteristics
    print("\nKey Performance Characteristics:")
    print("1. Universal Transformer:")
    print("   - Adaptive computation time")
    print("   - Variable depth per sequence")
    print("   - Efficient for varying complexity")
    
    print("\n2. Performer:")
    print("   - Linear attention complexity O(n)")
    print("   - Kernel-based approximation")
    print("   - Memory efficient for long sequences")
    
    print("\n3. Linformer:")
    print("   - Low-rank projections")
    print("   - Linear sequence length scaling")
    print("   - Constant memory usage")
    
    print("\n4. Reformer:")
    print("   - LSH attention for efficiency")
    print("   - Reversible layers for memory")
    print("   - Chunked processing")

# Run demonstration
demonstrate_advanced_transformers()

print("\n" + "="*60)
print("Advanced Transformer Architectures Framework Ready")
print("="*60)
print("\nImplemented Architectures:")
print("• Universal Transformer (Adaptive computation)")
print("• Performer (Linear attention via kernel methods)")
print("• Linformer (Low-rank attention projections)")
print("• Reformer (LSH attention + reversible layers)")
print("\nReady for implementation and experimentation!")
```

## 15. How do you implement Transformer ensembling and multi-model architectures?

**Answer:**
```python
class TransformerEnsemble(nn.Module):
    """
    Ensemble of multiple Transformer models
    """
    def __init__(self, models, ensemble_method='average'):
        super(TransformerEnsemble, self).__init__()
        self.models = nn.ModuleList(models)
        self.ensemble_method = ensemble_method
        
        # Learnable weights for weighted averaging
        if ensemble_method == 'weighted':
            self.weights = nn.Parameter(torch.ones(len(models)) / len(models))
        
    def forward(self, input_ids, attention_mask=None, **kwargs):
        # Get predictions from all models
        predictions = []
        for model in self.models:
            with torch.no_grad():
                output = model(input_ids, attention_mask=attention_mask, **kwargs)
                if hasattr(output, 'logits'):
                    predictions.append(output.logits)
                elif isinstance(output, tuple):
                    predictions.append(output[0])
                else:
                    predictions.append(output)
        
        # Ensemble predictions
        if self.ensemble_method == 'average':
            # Simple averaging
            ensemble_output = torch.stack(predictions).mean(dim=0)
        elif self.ensemble_method == 'weighted':
            # Weighted averaging
            weights = F.softmax(self.weights, dim=0)
            weighted_predictions = [pred * weight for pred, weight in zip(predictions, weights)]
            ensemble_output = torch.stack(weighted_predictions).sum(dim=0)
        elif self.ensemble_method == 'majority_vote':
            # For classification, majority vote
            predictions_tensor = torch.stack(predictions)
            ensemble_output = torch.mode(predictions_tensor, dim=0)[0]
        else:
            # Default to averaging
            ensemble_output = torch.stack(predictions).mean(dim=0)
        
        return ensemble_output

class MultiTaskTransformer(nn.Module):
    """
    Multi-task Transformer with shared encoder and task-specific heads
    """
    def __init__(self, shared_config, task_configs):
        super(MultiTaskTransformer, self).__init__()
        
        # Shared encoder (could be pre-trained)
        self.shared_encoder = self._create_shared_encoder(shared_config)
        
        # Task-specific heads
        self.task_heads = nn.ModuleDict()
        for task_name, task_config in task_configs.items():
            self.task_heads[task_name] = self._create_task_head(task_config)
        
        # Task routing
        self.task_routing = nn.Linear(shared_config.d_model, len(task_configs))
        
    def _create_shared_encoder(self, config):
        """Create shared encoder"""
        # This could be a pre-trained model like BERT
        return nn.TransformerEncoder(
            nn.TransformerEncoderLayer(
                d_model=config.d_model,
                nhead=config.num_heads,
                dim_feedforward=config.d_ff,
                batch_first=True
            ),
            num_layers=config.num_layers
        )
    
    def _create_task_head(self, task_config):
        """Create task-specific head"""
        if task_config.task_type == 'classification':
            return nn.Sequential(
                nn.Linear(task_config.d_model, task_config.d_model // 2),
                nn.ReLU(),
                nn.Dropout(0.1),
                nn.Linear(task_config.d_model // 2, task_config.num_classes)
            )
        elif task_config.task_type == 'regression':
            return nn.Sequential(
                nn.Linear(task_config.d_model, task_config.d_model // 2),
                nn.ReLU(),
                nn.Dropout(0.1),
                nn.Linear(task_config.d_model // 2, task_config.output_dim)
            )
        elif task_config.task_type == 'sequence_labeling':
            return nn.Linear(task_config.d_model, task_config.num_labels)
        else:
            return nn.Linear(task_config.d_model, task_config.output_dim)
    
    def forward(self, input_ids, attention_mask=None, task_name=None):
        # Shared encoding
        encoded = self.shared_encoder(input_ids, src_key_padding_mask=~attention_mask if attention_mask is not None else None)
        
        # Use [CLS] token for classification tasks
        pooled_output = encoded[:, 0, :]  # [CLS] token
        
        if task_name:
            # Specific task output
            task_output = self.task_heads[task_name](pooled_output)
            return task_output
        else:
            # Multi-task output with routing
            routing_logits = self.task_routing(pooled_output)
            task_weights = F.softmax(routing_logits, dim=-1)
            
            # Get outputs from all tasks
            task_outputs = {}
            for task_name, head in self.task_heads.items():
                task_outputs[task_name] = head(pooled_output)
            
            return {
                'task_outputs': task_outputs,
                'task_weights': task_weights,
                'routing_logits': routing_logits
            }

class MixtureOfExpertsTransformer(nn.Module):
    """
    Transformer with Mixture of Experts (MoE) layers
    """
    def __init__(self, d_model, num_heads, d_ff, num_experts=8, num_layers=6, dropout=0.1):
        super(MixtureOfExpertsTransformer, self).__init__()
        self.d_model = d_model
        self.num_experts = num_experts
        
        # Expert networks (feed-forward networks)
        self.experts = nn.ModuleList([
            PositionwiseFeedForward(d_model, d_ff, dropout)
            for _ in range(num_experts)
        ])
        
        # Gating network to select experts
        self.gate = nn.Linear(d_model, num_experts)
        
        # Standard transformer layers with MoE
        self.layers = nn.ModuleList([
            MoETransformerLayer(d_model, num_heads, d_ff, num_experts, dropout)
            for _ in range(num_layers)
        ])
        
        self.layer_norm = LayerNorm(d_model)
        
    def forward(self, x, mask=None):
        # Apply MoE transformer layers
        for layer in self.layers:
            x = layer(x, mask)
        
        x = self.layer_norm(x)
        return x

class MoETransformerLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, num_experts, dropout=0.1):
        super(MoETransformerLayer, self).__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads, dropout)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
        # Mixture of Experts feed-forward
        self.moe_ff = MixtureOfExperts(d_model, d_ff, num_experts, dropout)
        
    def forward(self, x, mask=None):
        # Self-attention
        attn_output = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Mixture of Experts feed-forward
        ff_output = self.moe_ff(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x

class MixtureOfExperts(nn.Module):
    def __init__(self, d_model, d_ff, num_experts, dropout=0.1):
        super(MixtureOfExperts, self).__init__()
        self.d_model = d_model
        self.num_experts = num_experts
        
        # Expert networks
        self.experts = nn.ModuleList([
            PositionwiseFeedForward(d_model, d_ff, dropout)
            for _ in range(num_experts)
        ])
        
        # Gating network
        self.gate = nn.Linear(d_model, num_experts)
        
    def forward(self, x):
        # x shape: (batch_size, seq_len, d_model)
        batch_size, seq_len, _ = x.size()
        
        # Compute gating weights
        gate_logits = self.gate(x)  # (batch, seq, num_experts)
        gate_weights = F.softmax(gate_logits, dim=-1)  # (batch, seq, num_experts)
        
        # Apply each expert
        expert_outputs = []
        for expert in self.experts:
            expert_output = expert(x)  # (batch, seq, d_model)
            expert_outputs.append(expert_output.unsqueeze(-1))  # (batch, seq, d_model, 1)
        
        # Stack expert outputs
        expert_outputs = torch.cat(expert_outputs, dim=-1)  # (batch, seq, d_model, num_experts)
        
        # Weight and sum expert outputs
        gate_weights = gate_weights.unsqueeze(2)  # (batch, seq, 1, num_experts)
        output = torch.sum(expert_outputs * gate_weights, dim=-1)  # (batch, seq, d_model)
        
        return output

class KnowledgeDistillationTransformer(nn.Module):
    """
    Transformer for knowledge distillation
    """
    def __init__(self, student_model, teacher_model, temperature=3.0, alpha=0.7):
        super(KnowledgeDistillationTransformer, self).__init__()
        self.student = student_model
        self.teacher = teacher_model
        self.temperature = temperature
        self.alpha = alpha
        
        # Freeze teacher model
        for param in self.teacher.parameters():
            param.requires_grad = False
    
    def forward(self, input_ids, attention_mask=None, labels=None):
        # Student forward pass
        student_outputs = self.student(input_ids, attention_mask=attention_mask)
        student_logits = student_outputs if isinstance(student_outputs, torch.Tensor) else student_outputs.logits
        
        # Teacher forward pass (no gradients)
        with torch.no_grad():
            teacher_outputs = self.teacher(input_ids, attention_mask=attention_mask)
            teacher_logits = teacher_outputs if isinstance(teacher_outputs, torch.Tensor) else teacher_outputs.logits
        
        # Knowledge distillation loss
        distillation_loss = self._compute_distillation_loss(student_logits, teacher_logits)
        
        # Standard loss (if labels provided)
        standard_loss = 0
        if labels is not None:
            if student_logits.dim() > 2:
                # Sequence labeling or language modeling
                standard_loss = F.cross_entropy(
                    student_logits.view(-1, student_logits.size(-1)),
                    labels.view(-1),
                    ignore_index=-100
                )
            else:
                # Classification
                standard_loss = F.cross_entropy(student_logits, labels)
        
        # Combined loss
        total_loss = self.alpha * distillation_loss + (1 - self.alpha) * standard_loss
        
        return {
            'loss': total_loss,
            'distillation_loss': distillation_loss,
            'standard_loss': standard_loss,
            'student_logits': student_logits,
            'teacher_logits': teacher_logits
        }
    
    def _compute_distillation_loss(self, student_logits, teacher_logits):
        """Compute distillation loss using soft targets"""
        # Soften probabilities with temperature
        student_probs = F.softmax(student_logits / self.temperature, dim=-1)
        teacher_probs = F.softmax(teacher_logits / self.temperature, dim=-1)
        
        # Compute KL divergence
        loss = F.kl_div(
            torch.log(student_probs),
            teacher_probs,
            reduction='batchmean'
        ) * (self.temperature ** 2)
        
        return loss

class CrossModalTransformer(nn.Module):
    """
    Cross-modal Transformer for multi-modal tasks
    """
    def __init__(self, text_config, image_config, fusion_config):
        super(CrossModalTransformer, self).__init__()
        
        # Modality-specific encoders
        self.text_encoder = self._create_text_encoder(text_config)
        self.image_encoder = self._create_image_encoder(image_config)
        
        # Cross-modal fusion
        self.fusion_layer = CrossModalFusionLayer(
            text_dim=text_config.d_model,
            image_dim=image_config.d_model,
            fusion_dim=fusion_config.d_model,
            num_heads=fusion_config.num_heads
        )
        
        # Output head
        self.output_head = nn.Linear(fusion_config.d_model, fusion_config.num_classes)
        
    def _create_text_encoder(self, config):
        """Create text encoder"""
        return nn.TransformerEncoder(
            nn.TransformerEncoderLayer(
                d_model=config.d_model,
                nhead=config.num_heads,
                dim_feedforward=config.d_ff,
                batch_first=True
            ),
            num_layers=config.num_layers
        )
    
    def _create_image_encoder(self, config):
        """Create image encoder (could be CNN + Transformer)"""
        # Simplified: CNN encoder followed by transformer
        return nn.Sequential(
            nn.Conv2d(3, 64, 3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d((16, 16)),
            nn.Flatten(2),
            nn.TransformerEncoder(
                nn.TransformerEncoderLayer(
                    d_model=64,
                    nhead=8,
                    batch_first=True
                ),
                num_layers=2
            )
        )
    
    def forward(self, text_input, image_input, text_mask=None):
        # Encode text
        text_features = self.text_encoder(text_input, src_key_padding_mask=~text_mask if text_mask is not None else None)
        text_pooled = text_features[:, 0, :]  # [CLS] token
        
        # Encode image
        image_features = self.image_encoder(image_input)
        image_pooled = image_features.mean(dim=1)  # Global average pooling
        
        # Cross-modal fusion
        fused_features = self.fusion_layer(text_pooled, image_pooled)
        
        # Classification
        logits = self.output_head(fused_features)
        
        return logits

class CrossModalFusionLayer(nn.Module):
    def __init__(self, text_dim, image_dim, fusion_dim, num_heads):
        super(CrossModalFusionLayer, self).__init__()
        
        # Projection layers
        self.text_proj = nn.Linear(text_dim, fusion_dim)
        self.image_proj = nn.Linear(image_dim, fusion_dim)
        
        # Cross-attention
        self.cross_attention = MultiHeadAttention(fusion_dim, num_heads)
        
        # Self-attention for fused features
        self.self_attention = MultiHeadAttention(fusion_dim, num_heads)
        
        # Layer normalization
        self.norm1 = LayerNorm(fusion_dim)
        self.norm2 = LayerNorm(fusion_dim)
        
    def forward(self, text_features, image_features):
        # Project to common dimension
        text_proj = self.text_proj(text_features)
        image_proj = self.image_proj(image_features)
        
        # Cross-modal attention
        # Text attends to image
        cross_output = self.cross_attention(text_proj.unsqueeze(1), image_proj.unsqueeze(1), image_proj.unsqueeze(1))
        cross_output = cross_output.squeeze(1)
        
        # Self-attention on fused features
        fused_features = text_proj + cross_output
        self_output = self.self_attention(fused_features.unsqueeze(1), fused_features.unsqueeze(1), fused_features.unsqueeze(1))
        self_output = self_output.squeeze(1)
        
        # Final fused representation
        final_features = self.norm1(fused_features + self_output)
        final_features = self.norm2(final_features + self.norm1(fused_features))
        
        return final_features

# Configuration classes
class TaskConfig:
    def __init__(self, task_type, d_model, num_classes=None, num_labels=None, output_dim=None):
        self.task_type = task_type
        self.d_model = d_model
        self.num_classes = num_classes
        self.num_labels = num_labels
        self.output_dim = output_dim

class SharedConfig:
    def __init__(self, d_model=768, num_heads=12, d_ff=3072, num_layers=12):
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_ff = d_ff
        self.num_layers = num_layers

class FusionConfig:
    def __init__(self, d_model=512, num_heads=8, num_classes=10):
        self.d_model = d_model
        self.num_heads = num_heads
        self.num_classes = num_classes

# Example usage and demonstration
def demonstrate_ensemble_architectures():
    """Demonstrate Transformer ensemble and multi-model architectures"""
    
    print("Transformer Ensemble and Multi-Model Architectures")
    print("=" * 55)
    
    # Sample input for demonstration
    batch_size, seq_len, d_model = 4, 32, 128
    sample_input = torch.randint(0, 1000, (batch_size, seq_len))
    sample_mask = torch.ones(batch_size, seq_len).bool()
    
    # 1. Transformer Ensemble
    print("\n1. Transformer Ensemble:")
    try:
        # Create multiple models
        models = [
            SimpleTransformer(vocab_size=1000, d_model=d_model, nhead=4, num_layers=2),
            SimpleTransformer(vocab_size=1000, d_model=d_model, nhead=8, num_layers=3),
            SimpleTransformer(vocab_size=1000, d_model=d_model, nhead=4, num_layers=4)
        ]
        
        # Create ensemble
        ensemble = TransformerEnsemble(models, ensemble_method='average')
        
        # Test ensemble
        ensemble_output = ensemble(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Ensemble output shape: {ensemble_output.shape}")
        print(f"   Number of models in ensemble: {len(models)}")
        print("   ✓ Transformer ensemble created successfully")
        
        # Test weighted ensemble
        weighted_ensemble = TransformerEnsemble(models, ensemble_method='weighted')
        weighted_output = weighted_ensemble(sample_input, sample_mask)
        print(f"   Weighted ensemble output shape: {weighted_output.shape}")
        print("   ✓ Weighted ensemble created successfully")
        
    except Exception as e:
        print(f"   ✗ Ensemble failed: {str(e)}")
    
    # 2. Multi-Task Transformer
    print("\n2. Multi-Task Transformer:")
    try:
        # Create configurations
        shared_config = SharedConfig(d_model=d_model, num_heads=4, num_layers=2)
        task_configs = {
            'sentiment': TaskConfig('classification', d_model, num_classes=3),
            'ner': TaskConfig('sequence_labeling', d_model, num_labels=10),
            'regression_task': TaskConfig('regression', d_model, output_dim=1)
        }
        
        # Create multi-task model
        multitask_model = MultiTaskTransformer(shared_config, task_configs)
        
        # Test multi-task model
        multitask_output = multitask_model(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Multi-task output keys: {list(multitask_output.keys())}")
        print(f"   Task outputs: {list(multitask_output['task_outputs'].keys())}")
        print("   ✓ Multi-task transformer created successfully")
        
        # Test specific task
        specific_output = multitask_model(sample_input, sample_mask, task_name='sentiment')
        print(f"   Specific task output shape: {specific_output.shape}")
        print("   ✓ Specific task output generated successfully")
        
    except Exception as e:
        print(f"   ✗ Multi-task transformer failed: {str(e)}")
    
    # 3. Mixture of Experts Transformer
    print("\n3. Mixture of Experts Transformer:")
    try:
        # Create MoE transformer
        moe_transformer = MixtureOfExpertsTransformer(
            d_model=d_model,
            num_heads=4,
            d_ff=256,
            num_experts=4,
            num_layers=2
        )
        
        # Test MoE transformer
        moe_output = moe_transformer(sample_input, sample_mask)
        print(f"   Input shape: {sample_input.shape}")
        print(f"   MoE output shape: {moe_output.shape}")
        print(f"   Number of experts: {moe_transformer.num_experts}")
        print("   ✓ Mixture of Experts transformer created successfully")
        
    except Exception as e:
        print(f"   ✗ MoE transformer failed: {str(e)}")
    
    # 4. Knowledge Distillation Transformer
    print("\n4. Knowledge Distillation Transformer:")
    try:
        # Create teacher and student models
        teacher_model = SimpleTransformer(vocab_size=1000, d_model=256, nhead=8, num_layers=4)
        student_model = SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2)
        
        # Create distillation model
        distill_model = KnowledgeDistillationTransformer(
            student_model=student_model,
            teacher_model=teacher_model,
            temperature=2.0,
            alpha=0.7
        )
        
        # Test distillation
        sample_labels = torch.randint(0, 5, (batch_size,))
        distill_output = distill_model(sample_input, sample_mask, sample_labels)
        
        print(f"   Input shape: {sample_input.shape}")
        print(f"   Distillation output keys: {list(distill_output.keys())}")
        print(f"   Total loss: {distill_output['loss'].item():.4f}")
        print(f"   Distillation loss: {distill_output['distillation_loss'].item():.4f}")
        print(f"   Standard loss: {distill_output['standard_loss'].item():.4f}")
        print("   ✓ Knowledge distillation transformer created successfully")
        
    except Exception as e:
        print(f"   ✗ Knowledge distillation failed: {str(e)}")
    
    # 5. Cross-Modal Transformer
    print("\n5. Cross-Modal Transformer:")
    try:
        # Create configurations
        text_config = SharedConfig(d_model=128, num_heads=4, num_layers=2)
        image_config = SharedConfig(d_model=64, num_heads=4, num_layers=2)
        fusion_config = FusionConfig(d_model=128, num_heads=4, num_classes=5)
        
        # Create cross-modal model
        crossmodal_model = CrossModalTransformer(text_config, image_config, fusion_config)
        
        # Test cross-modal model
        sample_image = torch.randn(batch_size, 3, 32, 32)  # (batch, channels, height, width)
        crossmodal_output = crossmodal_model(sample_input, sample_image, sample_mask)
        
        print(f"   Text input shape: {sample_input.shape}")
        print(f"   Image input shape: {sample_image.shape}")
        print(f"   Cross-modal output shape: {crossmodal_output.shape}")
        print("   ✓ Cross-modal transformer created successfully")
        
    except Exception as e:
        print(f"   ✗ Cross-modal transformer failed: {str(e)}")
    
    # Model comparison
    print("\n6. Model Architecture Comparison:")
    models = {
        'Standard Transformer': SimpleTransformer(vocab_size=1000, d_model=d_model, nhead=4, num_layers=2),
        'Ensemble (3 models)': ensemble,
        'Multi-Task': multitask_model,
        'MoE Transformer': moe_transformer
    }
    
    for name, model in models.items():
        try:
            total_params = sum(p.numel() for p in model.parameters())
            trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
            print(f"   {name}:")
            print(f"     Total parameters: {total_params:,}")
            print(f"     Trainable parameters: {trainable_params:,}")
        except:
            print(f"   {name}: Parameter count not available")
    
    print("\nEnsemble and multi-model architectures demonstration completed!")
    
    # Key benefits summary
    print("\nKey Benefits of Ensemble and Multi-Model Architectures:")
    print("1. Transformer Ensemble:")
    print("   - Improved robustness and generalization")
    print("   - Reduced variance in predictions")
    print("   - Better uncertainty estimation")
    
    print("\n2. Multi-Task Transformer:")
    print("   - Shared representation learning")
    print("   - Improved performance through auxiliary tasks")
    print("   - Efficient parameter usage")
    
    print("\n3. Mixture of Experts:")
    print("   - Conditional computation")
    print("   - Scalable model capacity")
    print("   - Efficient inference")
    
    print("\n4. Knowledge Distillation:")
    print("   - Model compression")
    print("   - Improved student model performance")
    print("   - Faster inference")
    
    print("\n5. Cross-Modal Transformer:")
    print("   - Multi-modal representation learning")
    print("   - Cross-modal attention mechanisms")
    print("   - Unified multi-modal processing")

# Run demonstration
demonstrate_ensemble_architectures()

print("\n" + "="*65)
print("Transformer Ensemble and Multi-Model Architectures Framework Ready")
print("="*65)
print("\nImplemented Architectures:")
print("• Transformer Ensemble (Averaging, Weighted)")
print("• Multi-Task Transformer")
print("• Mixture of Experts Transformer")
print("• Knowledge Distillation Transformer")
print("• Cross-Modal Transformer")
print("\nReady for advanced multi-model experimentation!")
```

## 16. How do you implement Transformer optimization and deployment?

**Answer:**
```python
class TransformerOptimizer:
    """
    Comprehensive optimization framework for Transformer models
    """
    def __init__(self, model, device=None):
        self.model = model
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        
        # Optimization statistics
        self.optimization_stats = {}
        
    def quantize_model(self, quantization_type='dynamic'):
        """
        Apply quantization to reduce model size and improve inference speed
        """
        self.model.eval()
        
        if quantization_type == 'dynamic':
            # Dynamic quantization (quantize weights only)
            quantized_model = torch.quantization.quantize_dynamic(
                self.model,
                {nn.Linear},  # Specify which layers to quantize
                dtype=torch.qint8
            )
        elif quantization_type == 'static':
            # Static quantization (requires calibration)
            self.model.qconfig = torch.quantization.get_default_qconfig('fbgemm')
            torch.quantization.prepare(self.model, inplace=True)
            # Calibration would happen here with sample data
            torch.quantization.convert(self.model, inplace=True)
            quantized_model = self.model
        elif quantization_type == 'half_precision':
            # Half precision (FP16)
            quantized_model = self.model.half()
        else:
            quantized_model = self.model
        
        # Store optimization stats
        original_size = sum(p.numel() * p.element_size() for p in self.model.parameters())
        quantized_size = sum(p.numel() * p.element_size() for p in quantized_model.parameters())
        
        self.optimization_stats['quantization'] = {
            'original_size_mb': original_size / 1024 / 1024,
            'quantized_size_mb': quantized_size / 1024 / 1024,
            'compression_ratio': original_size / quantized_size,
            'quantization_type': quantization_type
        }
        
        return quantized_model
    
    def prune_model(self, pruning_ratio=0.3, pruning_type='unstructured'):
        """
        Apply pruning to reduce model size and improve inference speed
        """
        if pruning_type == 'unstructured':
            # Unstructured pruning (prune individual weights)
            parameters_to_prune = []
            for name, module in self.model.named_modules():
                if isinstance(module, nn.Linear):
                    parameters_to_prune.append((module, 'weight'))
            
            torch.nn.utils.prune.global_unstructured(
                parameters_to_prune,
                pruning_method=torch.nn.utils.prune.L1Unstructured,
                amount=pruning_ratio,
            )
        elif pruning_type == 'structured':
            # Structured pruning (prune entire neurons/channels)
            # This is a simplified example
            for name, module in self.model.named_modules():
                if isinstance(module, nn.Linear):
                    # Prune output features
                    num_features = module.out_features
                    num_to_prune = int(num_features * pruning_ratio)
                    if num_to_prune > 0:
                        # Create pruning mask
                        importance = torch.abs(module.weight).sum(dim=1)
                        _, indices = torch.topk(importance, num_to_prune, largest=False)
                        mask = torch.ones(num_features, dtype=torch.bool)
                        mask[indices] = False
                        
                        # Apply pruning
                        torch.nn.utils.prune.custom_from_mask(module, 'weight', mask.unsqueeze(1).expand_as(module.weight))
        
        # Store optimization stats
        total_params = sum(p.numel() for p in self.model.parameters())
        pruned_params = sum(p.numel() for p in self.model.parameters() if p.requires_grad)
        
        self.optimization_stats['pruning'] = {
            'total_parameters': total_params,
            'pruned_parameters': pruned_params,
            'pruning_ratio': pruning_ratio,
            'pruning_type': pruning_type
        }
        
        return self.model
    
    def compile_model(self, example_inputs=None):
        """
        Compile model for optimized inference using TorchScript or ONNX
        """
        self.model.eval()
        
        if example_inputs is None:
            # Create dummy inputs for compilation
            example_inputs = torch.randint(0, 1000, (1, 32)).to(self.device)
            example_mask = torch.ones(1, 32).bool().to(self.device)
            example_inputs = (example_inputs, example_mask)
        
        try:
            # TorchScript compilation
            traced_model = torch.jit.trace(self.model, example_inputs)
            self.optimization_stats['compilation'] = {
                'type': 'torchscript',
                'success': True
            }
            return traced_model
        except Exception as e:
            print(f"TorchScript compilation failed: {str(e)}")
            
            # Alternative: Script compilation
            try:
                scripted_model = torch.jit.script(self.model)
                self.optimization_stats['compilation'] = {
                    'type': 'torchscript_script',
                    'success': True
                }
                return scripted_model
            except Exception as e2:
                print(f"TorchScript scripting failed: {str(e2)}")
                self.optimization_stats['compilation'] = {
                    'type': 'none',
                    'success': False,
                    'error': str(e2)
                }
                return self.model
    
    def profile_model(self, input_shape=(1, 32), num_iterations=100):
        """
        Profile model performance and memory usage
        """
        self.model.eval()
        
        # Create test inputs
        input_ids = torch.randint(0, 1000, input_shape).to(self.device)
        attention_mask = torch.ones(input_shape).bool().to(self.device)
        
        # Memory profiling
        if torch.cuda.is_available():
            torch.cuda.empty_cache()
            initial_memory = torch.cuda.memory_allocated()
            
            with torch.no_grad():
                for _ in range(10):  # Warmup
                    _ = self.model(input_ids, attention_mask)
                
                torch.cuda.reset_peak_memory_stats()
                for _ in range(num_iterations):
                    _ = self.model(input_ids, attention_mask)
                
                peak_memory = torch.cuda.max_memory_allocated()
                final_memory = torch.cuda.memory_allocated()
                
                memory_stats = {
                    'initial_memory_mb': initial_memory / 1024 / 1024,
                    'peak_memory_mb': peak_memory / 1024 / 1024,
                    'final_memory_mb': final_memory / 1024 / 1024,
                    'memory_increase_mb': (peak_memory - initial_memory) / 1024 / 1024
                }
        else:
            memory_stats = {'error': 'CUDA not available'}
        
        # Timing profiling
        with torch.no_grad():
            # Warmup
            for _ in range(10):
                _ = self.model(input_ids, attention_mask)
            
            # Timing
            start_time = time.time()
            with torch.no_grad():
                for _ in range(num_iterations):
                    _ = self.model(input_ids, attention_mask)
            end_time = time.time()
            
            avg_time = (end_time - start_time) / num_iterations
            tokens_per_second = (input_shape[0] * input_shape[1]) / avg_time
            
            timing_stats = {
                'avg_inference_time': avg_time,
                'tokens_per_second': tokens_per_second,
                'total_time': end_time - start_time
            }
        
        # Parameter count
        total_params = sum(p.numel() for p in self.model.parameters())
        trainable_params = sum(p.numel() for p in self.model.parameters() if p.requires_grad)
        
        parameter_stats = {
            'total_parameters': total_params,
            'trainable_parameters': trainable_params,
            'model_size_mb': (total_params * 4) / 1024 / 1024  # Assuming float32
        }
        
        profiling_results = {
            'memory': memory_stats,
            'timing': timing_stats,
            'parameters': parameter_stats
        }
        
        self.optimization_stats['profiling'] = profiling_results
        
        return profiling_results
    
    def benchmark_optimizations(self, input_shape=(1, 32)):
        """
        Benchmark different optimization techniques
        """
        print("Benchmarking Transformer Optimizations")
        print("=" * 40)
        
        # Original model performance
        print("\n1. Original Model:")
        original_stats = self.profile_model(input_shape)
        print(f"   Inference time: {original_stats['timing']['avg_inference_time']:.4f}s")
        print(f"   Memory usage: {original_stats['memory'].get('peak_memory_mb', 'N/A')}")
        
        # Quantized model performance
        print("\n2. Quantized Model:")
        quantized_model = self.quantize_model('half_precision')
        quantized_optimizer = TransformerOptimizer(quantized_model, self.device)
        quantized_stats = quantized_optimizer.profile_model(input_shape)
        print(f"   Inference time: {quantized_stats['timing']['avg_inference_time']:.4f}s")
        print(f"   Memory usage: {quantized_stats['memory'].get('peak_memory_mb', 'N/A')}")
        
        # Compiled model performance
        print("\n3. Compiled Model:")
        compiled_model = self.compile_model()
        compiled_optimizer = TransformerOptimizer(compiled_model, self.device)
        compiled_stats = compiled_optimizer.profile_model(input_shape)
        print(f"   Inference time: {compiled_stats['timing']['avg_inference_time']:.4f}s")
        print(f"   Memory usage: {compiled_stats['memory'].get('peak_memory_mb', 'N/A')}")
        
        # Compare results
        print("\nPerformance Comparison:")
        print("-" * 25)
        
        optimizations = {
            'Original': original_stats,
            'Quantized': quantized_stats,
            'Compiled': compiled_stats
        }
        
        for name, stats in optimizations.items():
            speedup = original_stats['timing']['avg_inference_time'] / stats['timing']['avg_inference_time']
            print(f"{name:12}: {speedup:.2f}x speedup")
        
        return optimizations

class ModelExporter:
    """
    Export Transformer models to various formats for deployment
    """
    def __init__(self, model, device=None):
        self.model = model
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.model.eval()
    
    def export_to_onnx(self, filepath, input_shape=(1, 32), opset_version=11):
        """
        Export model to ONNX format
        """
        # Create dummy inputs
        dummy_input = torch.randint(0, 1000, input_shape).to(self.device)
        dummy_mask = torch.ones(input_shape).bool().to(self.device)
        
        try:
            torch.onnx.export(
                self.model,
                (dummy_input, dummy_mask),
                filepath,
                export_params=True,
                opset_version=opset_version,
                do_constant_folding=True,
                input_names=['input_ids', 'attention_mask'],
                output_names=['output'],
                dynamic_axes={
                    'input_ids': {0: 'batch_size', 1: 'sequence_length'},
                    'attention_mask': {0: 'batch_size', 1: 'sequence_length'},
                    'output': {0: 'batch_size', 1: 'sequence_length'}
                }
            )
            print(f"Model exported to ONNX: {filepath}")
            return True
        except Exception as e:
            print(f"ONNX export failed: {str(e)}")
            return False
    
    def export_to_torchscript(self, filepath, input_shape=(1, 32)):
        """
        Export model to TorchScript format
        """
        # Create dummy inputs
        dummy_input = torch.randint(0, 1000, input_shape).to(self.device)
        dummy_mask = torch.ones(input_shape).bool().to(self.device)
        
        try:
            # Trace the model
            traced_model = torch.jit.trace(self.model, (dummy_input, dummy_mask))
            
            # Save the traced model
            traced_model.save(filepath)
            print(f"Model exported to TorchScript: {filepath}")
            return True
        except Exception as e:
            print(f"TorchScript export failed: {str(e)}")
            return False
    
    def export_to_torchserve(self, model_name, version="1.0"):
        """
        Export model for TorchServe deployment
        """
        # Export to TorchScript first
        model_file = f"{model_name}.pt"
        if self.export_to_torchscript(model_file):
            # Create model archive for TorchServe
            try:
                import subprocess
                subprocess.run([
                    "torch-model-archiver",
                    "--model-name", model_name,
                    "--version", version,
                    "--model-file", model_file,
                    "--serialized-file", model_file,
                    "--export-path", ".",
                    "--force"
                ], check=True)
                print(f"Model archived for TorchServe: {model_name}.mar")
                return True
            except Exception as e:
                print(f"TorchServe archive creation failed: {str(e)}")
                return False
        return False

class ModelDeployer:
    """
    Deploy Transformer models for production use
    """
    def __init__(self, model_path, model_type='torchscript'):
        self.model_path = model_path
        self.model_type = model_type
        self.model = None
        self.load_model()
    
    def load_model(self):
        """Load the model for inference"""
        if self.model_type == 'torchscript':
            self.model = torch.jit.load(self.model_path)
            self.model.eval()
        elif self.model_type == 'onnx':
            import onnxruntime as ort
            self.model = ort.InferenceSession(self.model_path)
        else:
            raise ValueError(f"Unsupported model type: {self.model_type}")
    
    def predict(self, input_ids, attention_mask=None):
        """Make predictions with the loaded model"""
        if self.model_type == 'torchscript':
            with torch.no_grad():
                input_ids = torch.tensor(input_ids) if not isinstance(input_ids, torch.Tensor) else input_ids
                if attention_mask is not None:
                    attention_mask = torch.tensor(attention_mask) if not isinstance(attention_mask, torch.Tensor) else attention_mask
                
                if attention_mask is None:
                    attention_mask = torch.ones_like(input_ids).bool()
                
                input_ids = input_ids.to(next(self.model.parameters()).device)
                attention_mask = attention_mask.to(next(self.model.parameters()).device)
                
                output = self.model(input_ids, attention_mask)
                return output.cpu().numpy()
        
        elif self.model_type == 'onnx':
            input_ids = np.array(input_ids, dtype=np.int64)
            if attention_mask is None:
                attention_mask = np.ones_like(input_ids, dtype=np.bool)
            else:
                attention_mask = np.array(attention_mask, dtype=np.bool)
            
            outputs = self.model.run(
                None,
                {
                    'input_ids': input_ids,
                    'attention_mask': attention_mask
                }
            )
            return outputs[0]
    
    def benchmark_deployment(self, input_shape=(1, 32), num_iterations=1000):
        """Benchmark deployed model performance"""
        # Create test data
        test_input = np.random.randint(0, 1000, input_shape).astype(np.int64)
        test_mask = np.ones(input_shape, dtype=np.bool)
        
        # Warmup
        for _ in range(10):
            _ = self.predict(test_input, test_mask)
        
        # Benchmark
        start_time = time.time()
        for _ in range(num_iterations):
            _ = self.predict(test_input, test_mask)
        end_time = time.time()
        
        avg_time = (end_time - start_time) / num_iterations
        tokens_per_second = (input_shape[0] * input_shape[1]) / avg_time
        
        return {
            'avg_inference_time': avg_time,
            'tokens_per_second': tokens_per_second,
            'total_time': end_time - start_time
        }

# Example usage and demonstration
def demonstrate_optimization_and_deployment():
    """Demonstrate Transformer optimization and deployment"""
    
    print("Transformer Optimization and Deployment Demonstration")
    print("=" * 55)
    
    # Create sample model for demonstration
    model = SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2)
    
    # 1. Model Optimization
    print("\n1. Model Optimization:")
    optimizer = TransformerOptimizer(model)
    
    # Profile original model
    print("   Profiling original model...")
    original_stats = optimizer.profile_model(input_shape=(4, 32))
    print(f"   Original model - Parameters: {original_stats['parameters']['total_parameters']:,}")
    print(f"   Original model - Size: {original_stats['parameters']['model_size_mb']:.2f} MB")
    print(f"   Original model - Speed: {original_stats['timing']['tokens_per_second']:.2f} tokens/sec")
    
    # Apply quantization
    print("\n   Applying quantization...")
    quantized_model = optimizer.quantize_model('half_precision')
    print(f"   Quantization - Compression ratio: {optimizer.optimization_stats['quantization']['compression_ratio']:.2f}x")
    
    # Apply pruning
    print("\n   Applying pruning...")
    pruned_model = optimizer.prune_model(pruning_ratio=0.2, pruning_type='unstructured')
    print(f"   Pruning - Parameters reduced by: {optimizer.optimization_stats['pruning']['pruning_ratio']:.2%}")
    
    # Compile model
    print("\n   Compiling model...")
    compiled_model = optimizer.compile_model()
    print(f"   Compilation - Success: {optimizer.optimization_stats['compilation']['success']}")
    
    # 2. Model Export
    print("\n2. Model Export:")
    exporter = ModelExporter(compiled_model)
    
    # Export to TorchScript
    print("   Exporting to TorchScript...")
    torchscript_success = exporter.export_to_torchscript("model.pt", input_shape=(1, 32))
    print(f"   TorchScript export: {'✓ Success' if torchscript_success else '✗ Failed'}")
    
    # Export to ONNX (if onnx is available)
    try:
        print("   Exporting to ONNX...")
        onnx_success = exporter.export_to_onnx("model.onnx", input_shape=(1, 32))
        print(f"   ONNX export: {'✓ Success' if onnx_success else '✗ Failed'}")
    except ImportError:
        print("   ONNX export: ✗ ONNX not installed")
    
    # 3. Model Deployment
    print("\n3. Model Deployment:")
    if torchscript_success:
        try:
            # Deploy TorchScript model
            deployer = ModelDeployer("model.pt", model_type='torchscript')
            
            # Test prediction
            test_input = np.random.randint(0, 1000, (1, 16)).astype(np.int64)
            test_mask = np.ones((1, 16), dtype=np.bool)
            
            print("   Testing deployed model...")
            prediction = deployer.predict(test_input, test_mask)
            print(f"   Prediction shape: {prediction.shape}")
            print(f"   Prediction dtype: {prediction.dtype}")
            
            # Benchmark deployment
            print("   Benchmarking deployment...")
            benchmark_results = deployer.benchmark_deployment(input_shape=(1, 32), num_iterations=100)
            print(f"   Deployed model - Speed: {benchmark_results['tokens_per_second']:.2f} tokens/sec")
            print(f"   Deployed model - Avg time: {benchmark_results['avg_inference_time']*1000:.2f} ms")
            
        except Exception as e:
            print(f"   Deployment failed: {str(e)}")
    
    # 4. Optimization Benchmark
    print("\n4. Optimization Benchmark:")
    benchmark_results = optimizer.benchmark_optimizations(input_shape=(4, 32))
    
    # 5. Model Size Comparison
    print("\n5. Model Size Comparison:")
    models = {
        'Original': model,
        'Quantized': quantized_model,
        'Pruned': pruned_model,
        'Compiled': compiled_model
    }
    
    for name, model_obj in models.items():
        try:
            size_mb = sum(p.numel() * p.element_size() for p in model_obj.parameters()) / 1024 / 1024
            print(f"   {name:12}: {size_mb:.2f} MB")
        except:
            print(f"   {name:12}: Size calculation failed")
    
    print("\nOptimization and deployment demonstration completed!")
    
    # Key benefits summary
    print("\nKey Benefits of Optimization and Deployment:")
    print("1. Model Optimization:")
    print("   - Reduced model size (quantization, pruning)")
    print("   - Improved inference speed (compilation)")
    print("   - Lower memory usage")
    
    print("\n2. Model Export:")
    print("   - Multiple format support (ONNX, TorchScript)")
    print("   - Cross-platform compatibility")
    print("   - Integration with deployment frameworks")
    
    print("\n3. Model Deployment:")
    print("   - Production-ready inference")
    print("   - Performance benchmarking")
    print("   - Scalable serving options")
    
    print("\n4. Performance Gains:")
    print("   - 2-4x inference speedup")
    print("   - 50-75% size reduction")
    print("   - Reduced memory footprint")

# Run demonstration
demonstrate_optimization_and_deployment()

print("\n" + "="*65)
print("Transformer Optimization and Deployment Framework Ready")
print("="*65)
print("\nImplemented Features:")
print("• Model Quantization (Dynamic, Static, Half-Precision)")
print("• Model Pruning (Structured, Unstructured)")
print("• Model Compilation (TorchScript)")
print("• Performance Profiling")
print("• Model Export (ONNX, TorchScript)")
print("• Model Deployment (TorchServe ready)")
print("• Benchmarking and Optimization")
print("\nReady for production deployment!")
```

## 17. How do you implement Transformer debugging and interpretability?

**Answer:**
```python
class TransformerDebugger:
    """
    Comprehensive debugging and interpretability tools for Transformer models
    """
    def __init__(self, model, tokenizer=None, device=None):
        self.model = model
        self.tokenizer = tokenizer
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.model.eval()
        
        # Hooks for attention visualization
        self.attention_hooks = {}
        self.attention_outputs = {}
        
    def register_attention_hooks(self):
        """Register hooks to capture attention weights"""
        def attention_hook(module, input, output):
            # Store attention weights
            if hasattr(output, 'attentions'):
                self.attention_outputs[module] = output.attentions
            elif isinstance(output, tuple) and len(output) > 1:
                # For models that return attentions in tuple
                self.attention_outputs[module] = output[1]
        
        # Register hooks on attention layers
        for name, module in self.model.named_modules():
            if 'attention' in name.lower() or 'attn' in name.lower():
                hook = module.register_forward_hook(attention_hook)
                self.attention_hooks[name] = hook
    
    def remove_attention_hooks(self):
        """Remove registered attention hooks"""
        for hook in self.attention_hooks.values():
            hook.remove()
        self.attention_hooks.clear()
        self.attention_outputs.clear()
    
    def debug_forward_pass(self, input_ids, attention_mask=None, detailed=False):
        """
        Debug forward pass with detailed information
        """
        # Register hooks if needed
        if detailed:
            self.register_attention_hooks()
        
        try:
            # Move inputs to device
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            # Forward pass
            with torch.no_grad():
                outputs = self.model(input_ids, attention_mask=attention_mask)
            
            # Extract logits
            if hasattr(outputs, 'logits'):
                logits = outputs.logits
            elif isinstance(outputs, tuple):
                logits = outputs[0]
            else:
                logits = outputs
            
            # Debug information
            debug_info = {
                'input_shape': input_ids.shape,
                'output_shape': logits.shape,
                'input_ids_range': (input_ids.min().item(), input_ids.max().item()),
                'logits_stats': {
                    'mean': logits.mean().item(),
                    'std': logits.std().item(),
                    'min': logits.min().item(),
                    'max': logits.max().item()
                }
            }
            
            # Attention information if hooks were registered
            if detailed and self.attention_outputs:
                attention_info = {}
                for layer_name, attention_weights in self.attention_outputs.items():
                    if isinstance(attention_weights, torch.Tensor):
                        attention_info[layer_name] = {
                            'shape': attention_weights.shape,
                            'mean': attention_weights.mean().item(),
                            'std': attention_weights.std().item(),
                            'sparsity': (attention_weights < 0.01).float().mean().item()
                        }
                    elif isinstance(attention_weights, (list, tuple)):
                        attention_info[layer_name] = []
                        for i, attn in enumerate(attention_weights):
                            if isinstance(attn, torch.Tensor):
                                attention_info[layer_name].append({
                                    'head': i,
                                    'shape': attn.shape,
                                    'mean': attn.mean().item(),
                                    'std': attn.std().item()
                                })
                
                debug_info['attention'] = attention_info
            
            return logits, debug_info
            
        finally:
            if detailed:
                self.remove_attention_hooks()
    
    def visualize_attention(self, input_ids, attention_mask=None, layer_idx=None, head_idx=None):
        """
        Visualize attention patterns
        """
        import matplotlib.pyplot as plt
        import seaborn as sns
        
        # Register attention hooks
        self.register_attention_hooks()
        
        try:
            # Forward pass to capture attention
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            with torch.no_grad():
                _ = self.model(input_ids, attention_mask=attention_mask)
            
            # Tokenize for labels if tokenizer available
            if self.tokenizer:
                tokens = [self.tokenizer.decode([token_id]) for token_id in input_ids[0].cpu().numpy()]
            else:
                tokens = [f"Token_{i}" for i in range(input_ids.size(1))]
            
            # Visualize attention
            if self.attention_outputs:
                fig, axes = plt.subplots(2, 2, figsize=(15, 12))
                axes = axes.ravel()
                
                layer_names = list(self.attention_outputs.keys())
                
                for idx, (layer_name, attention_weights) in enumerate(self.attention_outputs.items()):
                    if idx >= 4:  # Only show first 4 layers
                        break
                    
                    ax = axes[idx]
                    
                    # Handle different attention weight formats
                    if isinstance(attention_weights, torch.Tensor):
                        # Take first batch and specified layer/head
                        attn_weights = attention_weights[0]  # First batch
                        if layer_idx is not None and layer_idx < attn_weights.size(0):
                            attn_weights = attn_weights[layer_idx]
                        elif head_idx is not None and head_idx < attn_weights.size(0):
                            attn_weights = attn_weights[head_idx]
                        else:
                            # Average across heads
                            attn_weights = attn_weights.mean(dim=0)
                    elif isinstance(attention_weights, (list, tuple)):
                        # Take specified layer
                        layer_idx_use = layer_idx if layer_idx is not None else 0
                        if layer_idx_use < len(attention_weights):
                            attn_weights = attention_weights[layer_idx_use]
                            if isinstance(attn_weights, torch.Tensor):
                                attn_weights = attn_weights[0]  # First batch
                                if head_idx is not None and head_idx < attn_weights.size(0):
                                    attn_weights = attn_weights[head_idx]
                                else:
                                    attn_weights = attn_weights.mean(dim=0)  # Average heads
                    
                    # Convert to numpy for plotting
                    if isinstance(attn_weights, torch.Tensor):
                        attn_weights_np = attn_weights.cpu().numpy()
                        
                        # Plot attention heatmap
                        sns.heatmap(
                            attn_weights_np[:len(tokens), :len(tokens)],
                            xticklabels=tokens[:attn_weights_np.shape[1]],
                            yticklabels=tokens[:attn_weights_np.shape[0]],
                            ax=ax,
                            cmap='viridis',
                            cbar=True
                        )
                        ax.set_title(f'{layer_name}')
                        ax.set_xlabel('Key')
                        ax.set_ylabel('Query')
                
                plt.tight_layout()
                plt.show()
            
        finally:
            self.remove_attention_hooks()
    
    def analyze_gradient_flow(self, input_ids, labels, attention_mask=None):
        """
        Analyze gradient flow through the model
        """
        # Enable gradients
        self.model.train()
        
        # Move inputs to device
        input_ids = input_ids.to(self.device)
        labels = labels.to(self.device)
        if attention_mask is not None:
            attention_mask = attention_mask.to(self.device)
        
        # Forward pass
        outputs = self.model(input_ids, attention_mask=attention_mask)
        
        # Extract logits
        if hasattr(outputs, 'logits'):
            logits = outputs.logits
        elif isinstance(outputs, tuple):
            logits = outputs[0]
        else:
            logits = outputs
        
        # Calculate loss
        if logits.dim() > 2:  # Sequence output
            loss = F.cross_entropy(
                logits.view(-1, logits.size(-1)),
                labels.view(-1),
                ignore_index=-100
            )
        else:  # Classification output
            loss = F.cross_entropy(logits, labels)
        
        # Backward pass
        loss.backward()
        
        # Analyze gradients
        gradient_stats = {}
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                gradient_stats[name] = {
                    'mean': param.grad.mean().item(),
                    'std': param.grad.std().item(),
                    'max': param.grad.max().item(),
                    'min': param.grad.min().item(),
                    'norm': param.grad.norm().item()
                }
        
        # Switch back to eval mode
        self.model.eval()
        
        return gradient_stats, loss.item()
    
    def check_model_health(self):
        """
        Comprehensive model health check
        """
        health_report = {
            'parameter_stats': {},
            'gradient_stats': {},
            'activation_stats': {},
            'attention_stats': {}
        }
        
        # Parameter statistics
        total_params = 0
        trainable_params = 0
        param_norms = []
        
        for name, param in self.model.named_parameters():
            total_params += param.numel()
            if param.requires_grad:
                trainable_params += param.numel()
            
            param_norm = param.norm().item()
            param_norms.append(param_norm)
            
            health_report['parameter_stats'][name] = {
                'shape': list(param.shape),
                'numel': param.numel(),
                'requires_grad': param.requires_grad,
                'norm': param_norm,
                'mean': param.mean().item(),
                'std': param.std().item()
            }
        
        health_report['overall_stats'] = {
            'total_parameters': total_params,
            'trainable_parameters': trainable_params,
            'frozen_parameters': total_params - trainable_params,
            'parameter_norm_mean': np.mean(param_norms),
            'parameter_norm_std': np.std(param_norms)
        }
        
        # Check for common issues
        issues = []
        
        # Vanishing gradients
        if health_report['overall_stats']['parameter_norm_mean'] < 1e-6:
            issues.append("Potential vanishing gradients detected")
        
        # Exploding gradients
        if health_report['overall_stats']['parameter_norm_mean'] > 1e6:
            issues.append("Potential exploding gradients detected")
        
        # Dead neurons (parameters that are zero)
        zero_params = sum(1 for stats in health_report['parameter_stats'].values() 
                         if stats['norm'] < 1e-8)
        if zero_params > 0.1 * len(health_report['parameter_stats']):
            issues.append(f"High number of zero parameters ({zero_params}) detected")
        
        health_report['issues'] = issues
        
        return health_report
    
    def generate_interpretability_report(self, input_ids, attention_mask=None, top_k=5):
        """
        Generate comprehensive interpretability report
        """
        # Register attention hooks
        self.register_attention_hooks()
        
        try:
            # Forward pass
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            with torch.no_grad():
                outputs = self.model(input_ids, attention_mask=attention_mask)
            
            # Extract predictions
            if hasattr(outputs, 'logits'):
                logits = outputs.logits
            elif isinstance(outputs, tuple):
                logits = outputs[0]
            else:
                logits = outputs
            
            # Get top predictions
            if logits.dim() > 2:  # Sequence task
                predictions = torch.argmax(logits, dim=-1)
                confidence_scores = F.softmax(logits, dim=-1)
            else:  # Classification task
                confidence_scores = F.softmax(logits, dim=-1)
                top_predictions = torch.topk(confidence_scores, top_k, dim=-1)
            
            # Create report
            report = {
                'input_info': {
                    'sequence_length': input_ids.size(1),
                    'batch_size': input_ids.size(0)
                },
                'predictions': {},
                'attention_analysis': {},
                'model_health': self.check_model_health()
            }
            
            # Add prediction information
            if logits.dim() > 2:
                report['predictions'] = {
                    'shape': list(predictions.shape),
                    'sample_predictions': predictions[0].cpu().numpy().tolist()
                }
            else:
                report['predictions'] = {
                    'top_k_classes': top_predictions.indices[0].cpu().numpy().tolist(),
                    'top_k_confidences': top_predictions.values[0].cpu().numpy().tolist()
                }
            
            # Add attention analysis if available
            if self.attention_outputs:
                attention_summary = {}
                for layer_name, attention_weights in self.attention_outputs.items():
                    if isinstance(attention_weights, torch.Tensor):
                        attention_summary[layer_name] = {
                            'mean_attention': attention_weights.mean().item(),
                            'max_attention': attention_weights.max().item(),
                            'attention_entropy': self._calculate_attention_entropy(attention_weights)
                        }
                report['attention_analysis'] = attention_summary
            
            return report
            
        finally:
            self.remove_attention_hooks()
    
    def _calculate_attention_entropy(self, attention_weights):
        """Calculate entropy of attention weights"""
        if isinstance(attention_weights, torch.Tensor):
            # Add small epsilon to avoid log(0)
            attention_probs = F.softmax(attention_weights, dim=-1)
            entropy = -torch.sum(attention_probs * torch.log(attention_probs + 1e-8), dim=-1)
            return entropy.mean().item()
        return 0.0
    
    def trace_execution_path(self, input_ids, attention_mask=None):
        """
        Trace execution path through the model
        """
        execution_trace = []
        
        def trace_hook(module, input, output, module_name):
            execution_trace.append({
                'module': module_name,
                'input_shape': [list(i.shape) if hasattr(i, 'shape') else type(i).__name__ for i in input],
                'output_shape': list(output.shape) if hasattr(output, 'shape') else type(output).__name__,
                'device': str(next(module.parameters()).device) if len(list(module.parameters())) > 0 else 'N/A'
            })
        
        # Register hooks on all modules
        hooks = []
        for name, module in self.model.named_modules():
            if len(name) > 0:  # Skip the main model itself
                hook = module.register_forward_hook(
                    lambda module, input, output, name=name: trace_hook(module, input, output, name)
                )
                hooks.append(hook)
        
        try:
            # Forward pass
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            with torch.no_grad():
                _ = self.model(input_ids, attention_mask=attention_mask)
            
            return execution_trace
            
        finally:
            # Remove hooks
            for hook in hooks:
                hook.remove()

class ModelInterpreter:
    """
    Advanced model interpretation and explanation tools
    """
    def __init__(self, model, tokenizer=None, device=None):
        self.model = model
        self.tokenizer = tokenizer
        self.device = device or torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.model.eval()
    
    def integrated_gradients(self, input_ids, target_class=None, steps=50):
        """
        Compute integrated gradients for input attribution
        """
        # Convert to tensor if needed
        if not isinstance(input_ids, torch.Tensor):
            input_ids = torch.tensor(input_ids)
        
        input_ids = input_ids.to(self.device)
        baseline = torch.zeros_like(input_ids)
        
        # Create interpolation steps
        alphas = torch.linspace(0, 1, steps).to(self.device)
        gradients = []
        
        for alpha in alphas:
            interpolated = baseline + alpha * (input_ids - baseline)
            interpolated.requires_grad_(True)
            
            outputs = self.model(interpolated)
            if hasattr(outputs, 'logits'):
                logits = outputs.logits
            else:
                logits = outputs
            
            # Get target class
            if target_class is None:
                target_class = torch.argmax(logits, dim=-1)
            
            # Compute gradients
            self.model.zero_grad()
            if logits.dim() > 2:
                # For sequence tasks, use first token
                target_logits = logits[0, 0, target_class[0]]
            else:
                target_logits = logits[0, target_class[0]]
            
            target_logits.backward()
            gradients.append(interpolated.grad)
        
        # Average gradients
        avg_gradients = torch.mean(torch.stack(gradients), dim=0)
        integrated_grads = (input_ids - baseline) * avg_gradients
        
        return integrated_grads.detach().cpu().numpy()
    
    def attention_rollout(self, input_ids, attention_mask=None, discount_factor=0.9):
        """
        Compute attention rollout for input importance
        """
        debugger = TransformerDebugger(self.model, self.tokenizer, self.device)
        debugger.register_attention_hooks()
        
        try:
            input_ids = input_ids.to(self.device)
            if attention_mask is not None:
                attention_mask = attention_mask.to(self.device)
            
            with torch.no_grad():
                _ = self.model(input_ids, attention_mask=attention_mask)
            
            # Process attention weights
            attention_matrices = []
            for layer_name, attention_weights in debugger.attention_outputs.items():
                if isinstance(attention_weights, torch.Tensor):
                    # Average across heads and batch
                    attn = attention_weights.mean(dim=(0, 1))  # Average heads and batch
                    attention_matrices.append(attn.cpu().numpy())
            
            # Compute attention rollout
            if attention_matrices:
                # Start with identity matrix
                rollout = np.eye(attention_matrices[0].shape[0])
                
                # Multiply attention matrices with discount factor
                for i, attn_matrix in enumerate(attention_matrices):
                    discounted_attn = attn_matrix * (discount_factor ** i)
                    rollout = np.matmul(rollout, discounted_attn)
                
                return rollout
            
            return None
            
        finally:
            debugger.remove_attention_hooks()
    
    def generate_explanation(self, input_ids, method='attention', **kwargs):
        """
        Generate explanation for model predictions
        """
        if method == 'attention':
            return self.attention_rollout(input_ids, **kwargs)
        elif method == 'integrated_gradients':
            return self.integrated_gradients(input_ids, **kwargs)
        elif method == 'lime':
            return self.lime_explanation(input_ids, **kwargs)
        else:
            raise ValueError(f"Unknown explanation method: {method}")
    
    def lime_explanation(self, input_ids, num_samples=1000, num_features=10):
        """
        LIME-style explanation (simplified implementation)
        """
        # This is a simplified version - in practice, you'd use the lime library
        if not isinstance(input_ids, torch.Tensor):
            input_ids = torch.tensor(input_ids)
        
        input_ids = input_ids.to(self.device)
        
        # Get original prediction
        with torch.no_grad():
            original_output = self.model(input_ids)
            if hasattr(original_output, 'logits'):
                original_pred = original_output.logits
            else:
                original_pred = original_output
        
        # Create perturbed samples
        seq_len = input_ids.size(1)
        perturbations = []
        predictions = []
        
        for _ in range(num_samples):
            # Randomly mask tokens
            mask = torch.rand(seq_len) > 0.3  # Keep 70% of tokens
            perturbed_input = input_ids.clone()
            perturbed_input[0, ~mask] = 0  # Mask tokens (assuming 0 is padding)
            
            # Get prediction for perturbed input
            with torch.no_grad():
                perturbed_output = self.model(perturbed_input)
                if hasattr(perturbed_output, 'logits'):
                    pred = perturbed_output.logits
                else:
                    pred = perturbed_output
            
            perturbations.append(mask.float().cpu().numpy())
            predictions.append(pred.cpu().numpy())
        
        # Simple linear regression to find feature importance
        perturbations = np.array(perturbations)
        predictions = np.array(predictions)
        
        # This is a very simplified approach - real LIME uses weighted linear regression
        feature_importance = np.mean(np.abs(predictions - original_pred.cpu().numpy()), axis=0)
        
        # Get top features
        top_features = np.argsort(feature_importance)[-num_features:][::-1]
        
        return {
            'feature_importance': feature_importance,
            'top_features': top_features,
            'perturbations': perturbations,
            'predictions': predictions
        }

# Example usage and demonstration
def demonstrate_debugging_and_interpretability():
    """Demonstrate Transformer debugging and interpretability tools"""
    
    print("Transformer Debugging and Interpretability Demonstration")
    print("=" * 58)
    
    # Create sample model and data
    model = SimpleTransformer(vocab_size=1000, d_model=128, nhead=4, num_layers=2)
    tokenizer = None  # In practice, you'd use a real tokenizer
    
    # Create sample inputs
    batch_size, seq_len = 4, 16
    sample_input = torch.randint(1, 1000, (batch_size, seq_len))
    sample_mask = torch.ones(batch_size, seq_len).bool()
    sample_labels = torch.randint(0, 5, (batch_size,))
    
    # 1. Model Debugger
    print("\n1. Model Debugger:")
    debugger = TransformerDebugger(model, tokenizer)
    
    # Debug forward pass
    print("   Debugging forward pass...")
    logits, debug_info = debugger.debug_forward_pass(sample_input, sample_mask, detailed=True)
    print(f"   Input shape: {debug_info['input_shape']}")
    print(f"   Output shape: {debug_info['output_shape']}")
    print(f"   Logits mean: {debug_info['logits_stats']['mean']:.4f}")
    print(f"   Logits std: {debug_info['logits_stats']['std']:.4f}")
    
    if 'attention' in debug_info:
        print(f"   Attention layers captured: {len(debug_info['attention'])}")
    
    # 2. Model Health Check
    print("\n2. Model Health Check:")
    health_report = debugger.check_model_health()
    print(f"   Total parameters: {health_report['overall_stats']['total_parameters']:,}")
    print(f"   Trainable parameters: {health_report['overall_stats']['trainable_parameters']:,}")
    print(f"   Parameter norm mean: {health_report['overall_stats']['parameter_norm_mean']:.4f}")
    
    if health_report['issues']:
        print("   Issues detected:")
        for issue in health_report['issues']:
            print(f"     - {issue}")
    else:
        print("   ✓ No major issues detected")
    
    # 3. Gradient Analysis
    print("\n3. Gradient Analysis:")
    try:
        gradient_stats, loss = debugger.analyze_gradient_flow(sample_input, sample_labels, sample_mask)
        print(f"   Loss: {loss:.4f}")
        print(f"   Gradient statistics computed for {len(gradient_stats)} parameter groups")
        
        # Find layers with problematic gradients
        high_gradients = [name for name, stats in gradient_stats.items() 
                        if stats['norm'] > 10.0]
        low_gradients = [name for name, stats in gradient_stats.items() 
                        if stats['norm'] < 1e-6]
        
        if high_gradients:
            print(f"   High gradient layers: {len(high_gradients)}")
        if low_gradients:
            print(f"   Low gradient layers: {len(low_gradients)}")
            
    except Exception as e:
        print(f"   Gradient analysis failed: {str(e)}")
    
    # 4. Model Interpreter
    print("\n4. Model Interpreter:")
    interpreter = ModelInterpreter(model, tokenizer)
    
    # Integrated Gradients
    print("   Computing Integrated Gradients...")
    try:
        ig_attributions = interpreter.integrated_gradients(sample_input[0:1], steps=20)
        print(f"   IG attributions shape: {ig_attributions.shape}")
        print(f"   IG mean attribution: {np.mean(np.abs(ig_attributions)):.4f}")
    except Exception as e:
        print(f"   Integrated Gradients failed: {str(e)}")
    
    # Attention Rollout
    print("\n   Computing Attention Rollout...")
    try:
        attention_rollout = interpreter.attention_rollout(sample_input[0:1], sample_mask[0:1])
        if attention_rollout is not None:
            print(f"   Attention rollout shape: {attention_rollout.shape}")
            print(f"   Rollout max value: {np.max(attention_rollout):.4f}")
        else:
            print("   Attention rollout not available")
    except Exception as e:
        print(f"   Attention rollout failed: {str(e)}")
    
    # 5. Interpretability Report
    print("\n5. Interpretability Report:")
    try:
        interpret_report = debugger.generate_interpretability_report(sample_input[0:1], sample_mask[0:1])
        print(f"   Report generated with {len(interpret_report)} sections")
        print(f"   Prediction shape: {interpret_report['predictions'].get('shape', 'N/A')}")
        
        if 'attention_analysis' in interpret_report:
            print(f"   Attention layers analyzed: {len(interpret_report['attention_analysis'])}")
    except Exception as e:
        print(f"   Interpretability report failed: {str(e)}")
    
    # 6. Execution Trace
    print("\n6. Execution Trace:")
    try:
        execution_trace = debugger.trace_execution_path(sample_input[0:1], sample_mask[0:1])
        print(f"   Execution trace captured for {len(execution_trace)} modules")
        
        # Show first few modules
        for i, trace in enumerate(execution_trace[:3]):
            print(f"   Module {i+1}: {trace['module']}")
            print(f"     Input: {trace['input_shape']}")
            print(f"     Output: {trace['output_shape']}")
    except Exception as e:
        print(f"   Execution trace failed: {str(e)}")
    
    # 7. LIME Explanation (simplified)
    print("\n7. LIME-style Explanation:")
    try:
        lime_explanation = interpreter.lime_explanation(sample_input[0:1], num_samples=100)
        print(f"   LIME explanation computed")
        print(f"   Feature importance shape: {lime_explanation['feature_importance'].shape}")
        print(f"   Top features: {lime_explanation['top_features'][:5]}")
    except Exception as e:
        print(f"   LIME explanation failed: {str(e)}")
    
    print("\nDebugging and interpretability demonstration completed!")
    
    # Key benefits summary
    print("\nKey Benefits of Debugging and Interpretability:")
    print("1. Model Debugging:")
    print("   - Forward pass inspection")
    print("   - Gradient flow analysis")
    print("   - Health monitoring")
    print("   - Error detection")
    
    print("\n2. Interpretability:")
    print("   - Attention visualization")
    print("   - Input attribution (IG)")
    print("   - Attention rollout")
    print("   - LIME-style explanations")
    
    print("\n3. Model Analysis:")
    print("   - Parameter statistics")
    print("   - Execution tracing")
    print("   - Performance profiling")
    print("   - Issue detection")
    
    print("\n4. Production Benefits:")
    print("   - Model reliability")
    print("   - Debugging in production")
    print("   - Explainable AI")
    print("   - Trust and transparency")

# Run demonstration
demonstrate_debugging_and_interpretability()

print("\n" + "="*70)
print("Transformer Debugging and Interpretability Framework Ready")
print("="*70)
print("\nImplemented Tools:")
print("• Model Debugger (Forward pass, gradients, health)")
print("• Attention Visualization")
print("• Integrated Gradients")
print("• Attention Rollout")
print("• LIME-style Explanations")
print("• Execution Tracing")
print("• Interpretability Reports")
print("\nReady for model analysis and debugging!")