# Generative Models in PyTorch - 30+ Questions and Answers

## 1. What are generative models and what are the main types?

**Answer:** Generative models are machine learning models that learn the underlying distribution of data to generate new, similar data samples. The main types include:

1. **Variational Autoencoders (VAEs)**: Learn a probabilistic latent space and generate by sampling from it
2. **Generative Adversarial Networks (GANs)**: Use two competing networks (generator and discriminator)
3. **Normalizing Flows**: Transform simple distributions to complex ones through invertible transformations
4. **Diffusion Models**: Gradually denoise data from random noise through iterative processes
5. **Autoregressive Models**: Generate sequences token by token based on previous tokens
6. **Energy-Based Models**: Define energy functions to distinguish real vs generated samples

Each type has unique strengths: GANs for sharp images, VAEs for stable training, flows for exact likelihoods, and diffusion models for high-quality generation.

## 2. How do you implement a Variational Autoencoder (VAE) in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.distributions import Normal

class VAE(nn.Module):
    def __init__(self, input_dim, hidden_dim, latent_dim):
        super(VAE, self).__init__()
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.latent_dim = latent_dim
        
        # Encoder
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        
        # Latent space parameters
        self.fc_mu = nn.Linear(hidden_dim, latent_dim)
        self.fc_logvar = nn.Linear(hidden_dim, latent_dim)
        
        # Decoder
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, input_dim),
            nn.Sigmoid()  # For normalized image data
        )
    
    def encode(self, x):
        h = self.encoder(x)
        mu = self.fc_mu(h)
        logvar = self.fc_logvar(h)
        return mu, logvar
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def decode(self, z):
        return self.decoder(z)
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        recon_x = self.decode(z)
        return recon_x, mu, logvar

def vae_loss(recon_x, x, mu, logvar, beta=1.0):
    # Reconstruction loss
    recon_loss = F.binary_cross_entropy(recon_x, x, reduction='sum')
    
    # KL divergence loss
    kl_loss = -0.5 * torch.sum(1 + logvar - mu.pow(2) - logvar.exp())
    
    return recon_loss + beta * kl_loss, recon_loss, kl_loss

# Example usage
input_dim = 784  # 28x28 images flattened
vae = VAE(input_dim=input_dim, hidden_dim=400, latent_dim=20)

# Sample forward pass
sample_input = torch.randn(32, input_dim)
recon_x, mu, logvar = vae(sample_input)
loss, recon_loss, kl_loss = vae_loss(recon_x, sample_input, mu, logvar)

print(f"Input shape: {sample_input.shape}")
print(f"Reconstructed shape: {recon_x.shape}")
print(f"Loss: {loss.item():.4f}")
print(f"Reconstruction loss: {recon_loss.item():.4f}")
print(f"KL loss: {kl_loss.item():.4f}")
```

## 3. How do you implement a basic Generative Adversarial Network (GAN) in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F

class Generator(nn.Module):
    def __init__(self, noise_dim, img_dim):
        super(Generator, self).__init__()
        self.model = nn.Sequential(
            nn.Linear(noise_dim, 256),
            nn.LeakyReLU(0.2),
            nn.Linear(256, 512),
            nn.LeakyReLU(0.2),
            nn.Linear(512, 1024),
            nn.LeakyReLU(0.2),
            nn.Linear(1024, img_dim),
            nn.Tanh()  # Output in [-1, 1] range
        )
    
    def forward(self, x):
        return self.model(x)

class Discriminator(nn.Module):
    def __init__(self, img_dim):
        super(Discriminator, self).__init__()
        self.model = nn.Sequential(
            nn.Linear(img_dim, 1024),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            nn.Linear(1024, 512),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            nn.Linear(256, 1),
            nn.Sigmoid()  # Output probability of real vs fake
        )
    
    def forward(self, x):
        return self.model(x)

class GAN:
    def __init__(self, noise_dim, img_dim, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = Generator(noise_dim, img_dim).to(self.device)
        self.discriminator = Discriminator(img_dim).to(self.device)
        
        self.optimizer_G = optim.Adam(self.generator.parameters(), lr=lr, betas=(0.5, 0.999))
        self.optimizer_D = optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0.5, 0.999))
        
        self.criterion = nn.BCELoss()
        self.noise_dim = noise_dim
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1).to(self.device)
        fake_labels = torch.zeros(batch_size, 1).to(self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim).to(self.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# Example usage
noise_dim = 100
img_dim = 784  # Flattened 28x28 image
gan = GAN(noise_dim, img_dim)

# Sample training step
sample_real_data = torch.randn(32, img_dim)
d_loss, g_loss = gan.train_step(sample_real_data)

print(f"Discriminator loss: {d_loss:.4f}")
print(f"Generator loss: {g_loss:.4f}")

# Generate sample images
with torch.no_grad():
    noise = torch.randn(10, noise_dim).to(gan.device)
    generated_images = gan.generator(noise)
    print(f"Generated images shape: {generated_images.shape}")
```

## 4. How do you implement a Deep Convolutional GAN (DCGAN)?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DCGANGenerator(nn.Module):
    def __init__(self, noise_dim, ngf, nc):
        super(DCGANGenerator, self).__init__()
        self.main = nn.Sequential(
            # Input is Z, going into a convolution
            nn.ConvTranspose2d(noise_dim, ngf * 8, 4, 1, 0, bias=False),
            nn.BatchNorm2d(ngf * 8),
            nn.ReLU(True),
            # State size: (ngf*8) x 4 x 4
            nn.ConvTranspose2d(ngf * 8, ngf * 4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf * 4),
            nn.ReLU(True),
            # State size: (ngf*4) x 8 x 8
            nn.ConvTranspose2d(ngf * 4, ngf * 2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf * 2),
            nn.ReLU(True),
            # State size: (ngf*2) x 16 x 16
            nn.ConvTranspose2d(ngf * 2, ngf, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf),
            nn.ReLU(True),
            # State size: (ngf) x 32 x 32
            nn.ConvTranspose2d(ngf, nc, 4, 2, 1, bias=False),
            nn.Tanh()
            # State size: (nc) x 64 x 64
        )

    def forward(self, input):
        return self.main(input)

class DCGANDiscriminator(nn.Module):
    def __init__(self, nc, ndf):
        super(DCGANDiscriminator, self).__init__()
        self.main = nn.Sequential(
            # Input: (nc) x 64 x 64
            nn.Conv2d(nc, ndf, 4, 2, 1, bias=False),
            nn.LeakyReLU(0.2, inplace=True),
            # State size: (ndf) x 32 x 32
            nn.Conv2d(ndf, ndf * 2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 2),
            nn.LeakyReLU(0.2, inplace=True),
            # State size: (ndf*2) x 16 x 16
            nn.Conv2d(ndf * 2, ndf * 4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 4),
            nn.LeakyReLU(0.2, inplace=True),
            # State size: (ndf*4) x 8 x 8
            nn.Conv2d(ndf * 4, ndf * 8, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 8),
            nn.LeakyReLU(0.2, inplace=True),
            # State size: (ndf*8) x 4 x 4
            nn.Conv2d(ndf * 8, 1, 4, 1, 0, bias=False),
            nn.Sigmoid()
        )

    def forward(self, input):
        return self.main(input).view(-1, 1).squeeze(1)

class DCGAN:
    def __init__(self, noise_dim=100, img_channels=3, feature_maps_gen=64, feature_maps_disc=64):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = DCGANGenerator(noise_dim, feature_maps_gen, img_channels).to(self.device)
        self.discriminator = DCGANDiscriminator(img_channels, feature_maps_disc).to(self.device)
        
        # Initialize weights
        self.generator.apply(self.weights_init)
        self.discriminator.apply(self.weights_init)
        
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=0.0002, betas=(0.5, 0.999))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=0.0002, betas=(0.5, 0.999))
        
        self.criterion = nn.BCELoss()
        self.noise_dim = noise_dim
    
    def weights_init(self, m):
        """Initialize weights with mean=0, std=0.02"""
        classname = m.__class__.__name__
        if classname.find('Conv') != -1:
            nn.init.normal_(m.weight.data, 0.0, 0.02)
        elif classname.find('BatchNorm') != -1:
            nn.init.normal_(m.weight.data, 1.0, 0.02)
            nn.init.constant_(m.bias.data, 0)
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, device=self.device)
        fake_labels = torch.zeros(batch_size, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim, 1, 1, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# Example usage
dcgan = DCGAN(noise_dim=100, img_channels=3, feature_maps_gen=64, feature_maps_disc=64)

# Sample input: batch of 64x64 RGB images
sample_real_data = torch.randn(4, 3, 64, 64)
d_loss, g_loss = dcgan.train_step(sample_real_data)

print(f"DCGAN Discriminator loss: {d_loss:.4f}")
print(f"DCGAN Generator loss: {g_loss:.4f}")

# Generate sample images
with torch.no_grad():
    noise = torch.randn(5, 100, 1, 1, device=dcgan.device)
    generated_images = dcgan.generator(noise)
    print(f"DCGAN generated images shape: {generated_images.shape}")
```

## 5. How do you implement a Conditional GAN (cGAN)?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ConditionalGenerator(nn.Module):
    def __init__(self, noise_dim, num_classes, img_dim, hidden_dim=256):
        super(ConditionalGenerator, self).__init__()
        
        # Label embedding
        self.label_emb = nn.Embedding(num_classes, num_classes)
        
        self.model = nn.Sequential(
            nn.Linear(noise_dim + num_classes, hidden_dim),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim, hidden_dim * 2),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim * 2, hidden_dim * 4),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim * 4, img_dim),
            nn.Tanh()
        )
    
    def forward(self, noise, labels):
        # Embed labels
        label_emb = self.label_emb(labels)
        
        # Concatenate noise and labels
        gen_input = torch.cat([noise, label_emb], dim=1)
        
        return self.model(gen_input)

class ConditionalDiscriminator(nn.Module):
    def __init__(self, img_dim, num_classes, hidden_dim=256):
        super(ConditionalDiscriminator, self).__init__()
        
        # Label embedding
        self.label_emb = nn.Embedding(num_classes, num_classes)
        
        self.model = nn.Sequential(
            nn.Linear(img_dim + num_classes, hidden_dim * 4),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.4),
            nn.Linear(hidden_dim * 4, hidden_dim * 2),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.4),
            nn.Linear(hidden_dim * 2, hidden_dim),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.4),
            nn.Linear(hidden_dim, 1),
            nn.Sigmoid()
        )
    
    def forward(self, img, labels):
        # Embed labels
        label_emb = self.label_emb(labels)
        
        # Flatten image and concatenate with labels
        img_flat = img.view(img.size(0), -1)
        d_input = torch.cat([img_flat, label_emb], dim=1)
        
        return self.model(d_input)

class ConditionalGAN:
    def __init__(self, noise_dim, num_classes, img_dim, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = ConditionalGenerator(noise_dim, num_classes, img_dim).to(self.device)
        self.discriminator = ConditionalDiscriminator(img_dim, num_classes).to(self.device)
        
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0.5, 0.999))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0.5, 0.999))
        
        self.criterion = nn.BCELoss()
        self.noise_dim = noise_dim
    
    def train_step(self, real_data, labels):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data, labels)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim, device=self.device)
        fake_data = self.generator(noise, labels)
        fake_output = self.discriminator(fake_data.detach(), labels)
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data, labels)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()
    
    def generate_samples(self, num_samples, labels, noise=None):
        """Generate samples for specific classes"""
        if noise is None:
            noise = torch.randn(num_samples, self.noise_dim, device=self.device)
        
        labels = labels.to(self.device)
        
        with torch.no_grad():
            generated = self.generator(noise, labels)
        
        return generated

# Example usage
num_classes = 10
img_dim = 784  # For 28x28 images
noise_dim = 100

cgan = ConditionalGAN(noise_dim, num_classes, img_dim)

# Sample training data
sample_real_data = torch.randn(32, img_dim)
sample_labels = torch.randint(0, num_classes, (32,))

d_loss, g_loss = cgan.train_step(sample_real_data, sample_labels)

print(f"Conditional GAN Discriminator loss: {d_loss:.4f}")
print(f"Conditional GAN Generator loss: {g_loss:.4f}")

# Generate samples for specific classes
sample_labels_gen = torch.tensor([0, 1, 2, 3, 4, 5, 6, 7, 8, 9])
generated_samples = cgan.generate_samples(10, sample_labels_gen)
print(f"Generated samples shape: {generated_samples.shape}")
```

## 6. How do you implement a Deep Convolutional Conditional GAN?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DCGANConditionalGenerator(nn.Module):
    def __init__(self, noise_dim, num_classes, ngf=64, nc=3, img_size=64):
        super(DCGANConditionalGenerator, self).__init__()
        
        # Label embedding
        self.label_emb = nn.Embedding(num_classes, num_classes)
        self.img_size = img_size
        
        self.init_size = img_size // 16  # Initial size after convolutions
        self.l1 = nn.Sequential(nn.Linear(noise_dim + num_classes, ngf * 8 * self.init_size ** 2))
        
        self.conv_blocks = nn.Sequential(
            nn.BatchNorm2d(ngf * 8),
            nn.Upsample(scale_factor=2),
            nn.Conv2d(ngf * 8, ngf * 4, 3, stride=1, padding=1),
            nn.BatchNorm2d(ngf * 4),
            nn.ReLU(True),
            nn.Upsample(scale_factor=2),
            nn.Conv2d(ngf * 4, ngf * 2, 3, stride=1, padding=1),
            nn.BatchNorm2d(ngf * 2),
            nn.ReLU(True),
            nn.Upsample(scale_factor=2),
            nn.Conv2d(ngf * 2, ngf, 3, stride=1, padding=1),
            nn.BatchNorm2d(ngf),
            nn.ReLU(True),
            nn.Upsample(scale_factor=2),
            nn.Conv2d(ngf, nc, 3, stride=1, padding=1),
            nn.Tanh(),
        )

    def forward(self, noise, labels):
        # Embed labels
        label_emb = self.label_emb(labels)
        
        # Concatenate noise and labels
        gen_input = torch.cat([noise, label_emb], dim=1)
        
        out = self.l1(gen_input)
        out = out.view(out.shape[0], -1, self.init_size, self.init_size)
        img = self.conv_blocks(out)
        
        return img

class DCGANConditionalDiscriminator(nn.Module):
    def __init__(self, num_classes, ndf=64, nc=3, img_size=64):
        super(DCGANConditionalDiscriminator, self).__init__()
        
        # Label embedding
        self.label_emb = nn.Embedding(num_classes, num_classes)
        self.img_size = img_size
        self.init_size = img_size // 16
        
        # Image processing
        self.conv_blocks = nn.Sequential(
            nn.Conv2d(nc, ndf, 4, stride=2, padding=1),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Conv2d(ndf, ndf * 2, 4, stride=2, padding=1),
            nn.BatchNorm2d(ndf * 2),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Conv2d(ndf * 2, ndf * 4, 4, stride=2, padding=1),
            nn.BatchNorm2d(ndf * 4),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Conv2d(ndf * 4, ndf * 8, 4, stride=2, padding=1),
            nn.BatchNorm2d(ndf * 8),
            nn.LeakyReLU(0.2, inplace=True),
        )
        
        # Flatten and concatenate with label embedding
        self.ad_layer = nn.Sequential(
            nn.Linear(ndf * 8 * self.init_size ** 2 + num_classes, 1024),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Linear(1024, 1),
            nn.Sigmoid()
        )

    def forward(self, img, labels):
        # Process image
        img_flat = self.conv_blocks(img)
        img_flat = img_flat.view(img_flat.size(0), -1)
        
        # Embed labels
        label_emb = self.label_emb(labels)
        
        # Concatenate processed image and labels
        d_input = torch.cat([img_flat, label_emb], dim=1)
        
        validity = self.ad_layer(d_input)
        
        return validity

class DCGANConditional:
    def __init__(self, noise_dim=100, num_classes=10, img_channels=3, img_size=64, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = DCGANConditionalGenerator(noise_dim, num_classes, img_size=img_size).to(self.device)
        self.discriminator = DCGANConditionalDiscriminator(num_classes, img_size=img_size).to(self.device)
        
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0.5, 0.999))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0.5, 0.999))
        
        self.criterion = nn.BCELoss()
        self.noise_dim = noise_dim
    
    def train_step(self, real_data, labels):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data, labels)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim, device=self.device)
        fake_data = self.generator(noise, labels)
        fake_output = self.discriminator(fake_data.detach(), labels)
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data, labels)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()
    
    def generate_samples(self, num_samples, labels, noise=None):
        """Generate samples for specific classes"""
        if noise is None:
            noise = torch.randn(num_samples, self.noise_dim, device=self.device)
        
        labels = labels.to(self.device)
        
        with torch.no_grad():
            generated = self.generator(noise, labels)
        
        return generated

# Example usage
conditional_dcgan = DCGANConditional(
    noise_dim=100, 
    num_classes=10, 
    img_channels=3, 
    img_size=64, 
    lr=0.0002
)

# Sample training data
sample_real_data = torch.randn(4, 3, 64, 64)
sample_labels = torch.randint(0, 10, (4,))

d_loss, g_loss = conditional_dcgan.train_step(sample_real_data, sample_labels)

print(f"Conditional DCGAN Discriminator loss: {d_loss:.4f}")
print(f"Conditional DCGAN Generator loss: {g_loss:.4f}")

# Generate samples for specific classes
sample_labels_gen = torch.tensor([0, 1, 2, 3])
generated_samples = conditional_dcgan.generate_samples(4, sample_labels_gen)
print(f"Conditional DCGAN generated samples shape: {generated_samples.shape}")
```

## 7. How do you implement a Progressive Growing GAN (ProGAN)?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class EqualizedLinear(nn.Module):
    """Linear layer with equalized learning rate"""
    def __init__(self, in_dim, out_dim):
        super(EqualizedLinear, self).__init__()
        self.linear = nn.Linear(in_dim, out_dim)
        # Initialize weights with He initialization
        nn.init.kaiming_normal_(self.linear.weight)
        self.linear.bias.data.fill_(0)
        
        # Compute fan-in for normalization
        self.scale = (self.linear.weight.data.shape[1] ** 0.5)
    
    def forward(self, x):
        return self.linear(x * self.scale)

class EqualizedConv2d(nn.Module):
    """Conv2d layer with equalized learning rate"""
    def __init__(self, in_dim, out_dim, kernel_size, stride=1, padding=0):
        super(EqualizedConv2d, self).__init__()
        self.conv = nn.Conv2d(in_dim, out_dim, kernel_size, stride, padding)
        # Initialize weights with He initialization
        nn.init.kaiming_normal_(self.conv.weight)
        self.conv.bias.data.fill_(0)
        
        # Compute fan-in for normalization
        self.scale = (self.conv.weight.data.shape[1] * kernel_size ** 2) ** 0.5
    
    def forward(self, x):
        return self.conv(x * self.scale)

class PixelNorm(nn.Module):
    def __init__(self):
        super(PixelNorm, self).__init__()

    def forward(self, x):
        return x / torch.sqrt(torch.mean(x ** 2, dim=1, keepdim=True) + 1e-8)

class GenBlock(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(GenBlock, self).__init__()
        
        self.block = nn.Sequential(
            nn.Upsample(scale_factor=2),
            EqualizedConv2d(in_channels, out_channels, 3, 1, 1),
            nn.LeakyReLU(0.2),
            PixelNorm(),
            EqualizedConv2d(out_channels, out_channels, 3, 1, 1),
            nn.LeakyReLU(0.2),
            PixelNorm()
        )
    
    def forward(self, x):
        return self.block(x)

class Generator(nn.Module):
    def __init__(self, z_dim, n_features, img_channels=3):
        super(Generator, self).__init__()
        self.z_dim = z_dim
        self.n_features = n_features
        
        # Initial block
        self.initial = nn.Sequential(
            EqualizedLinear(z_dim, n_features * 16),
            nn.LeakyReLU(0.2),
            PixelNorm(),
            EqualizedLinear(n_features * 16, n_features * 16 * 4 * 4),
            nn.LeakyReLU(0.2),
            PixelNorm()
        )
        
        # Reshape to 4x4 image
        self.reshape = lambda x: x.view(x.size(0), -1, 4, 4)
        
        # Progressive blocks
        self.progressive_blocks = nn.ModuleList([
            GenBlock(n_features * 16, n_features * 16),  # 4x4 -> 8x8
            GenBlock(n_features * 16, n_features * 8),   # 8x8 -> 16x16
            GenBlock(n_features * 8, n_features * 4),    # 16x16 -> 32x32
            GenBlock(n_features * 4, n_features * 2),    # 32x32 -> 64x64
            GenBlock(n_features * 2, n_features)         # 64x64 -> 128x128
        ])
        
        # To RGB output layers for each resolution
        self.to_rgb_layers = nn.ModuleList([
            EqualizedConv2d(n_features * 16, img_channels, 1),  # 4x4
            EqualizedConv2d(n_features * 16, img_channels, 1),  # 8x8
            EqualizedConv2d(n_features * 8, img_channels, 1),   # 16x16
            EqualizedConv2d(n_features * 4, img_channels, 1),   # 32x32
            EqualizedConv2d(n_features * 2, img_channels, 1),   # 64x64
            EqualizedConv2d(n_features, img_channels, 1)        # 128x128
        ])
    
    def forward(self, x, alpha, progress):
        # Initial block
        x = self.initial(x)
        x = self.reshape(x)
        
        # Apply progressive blocks up to current progress
        for i in range(progress):
            x = self.progressive_blocks[i](x)
        
        # Apply to RGB layer for current resolution
        if progress == 0:
            out = self.to_rgb_layers[0](x)
        else:
            # Fade in between resolutions
            x_prev = self.progressive_blocks[progress-1](x)
            out_prev = self.to_rgb_layers[progress-1](x_prev)
            out_curr = self.to_rgb_layers[progress](x)
            out = alpha * out_curr + (1 - alpha) * F.interpolate(out_prev, scale_factor=2, mode='nearest')
        
        return out

class Discriminator(nn.Module):
    def __init__(self, n_features, img_channels=3):
        super(Discriminator, self).__init__()
        self.n_features = n_features
        
        # From RGB input layers for each resolution
        self.from_rgb_layers = nn.ModuleList([
            EqualizedConv2d(img_channels, n_features, 1),        # 128x128
            EqualizedConv2d(img_channels, n_features * 2, 1),    # 64x64
            EqualizedConv2d(img_channels, n_features * 4, 1),    # 32x32
            EqualizedConv2d(img_channels, n_features * 8, 1),    # 16x16
            EqualizedConv2d(img_channels, n_features * 16, 1),   # 8x8
            EqualizedConv2d(img_channels, n_features * 16, 1)    # 4x4
        ])
        
        # Progressive blocks (reversed from generator)
        self.progressive_blocks = nn.ModuleList([
            nn.Sequential(  # 4x4 -> 8x8
                EqualizedConv2d(n_features * 16, n_features * 16, 3, 1, 1),
                nn.LeakyReLU(0.2),
                EqualizedConv2d(n_features * 16, n_features * 16, 4, 1, 0),
                nn.LeakyReLU(0.2),
            ),
            nn.Sequential(  # 8x8 -> 16x16
                EqualizedConv2d(n_features * 16, n_features * 8, 3, 1, 1),
                nn.LeakyReLU(0.2),
                nn.AvgPool2d(2),
                EqualizedConv2d(n_features * 8, n_features * 16, 3, 1, 1),
                nn.LeakyReLU(0.2),
            ),
            nn.Sequential(  # 16x16 -> 32x32
                EqualizedConv2d(n_features * 8, n_features * 4, 3, 1, 1),
                nn.LeakyReLU(0.2),
                nn.AvgPool2d(2),
                EqualizedConv2d(n_features * 4, n_features * 8, 3, 1, 1),
                nn.LeakyReLU(0.2),
            ),
            nn.Sequential(  # 32x32 -> 64x64
                EqualizedConv2d(n_features * 4, n_features * 2, 3, 1, 1),
                nn.LeakyReLU(0.2),
                nn.AvgPool2d(2),
                EqualizedConv2d(n_features * 2, n_features * 4, 3, 1, 1),
                nn.LeakyReLU(0.2),
            ),
            nn.Sequential(  # 64x64 -> 128x128
                EqualizedConv2d(n_features * 2, n_features, 3, 1, 1),
                nn.LeakyReLU(0.2),
                nn.AvgPool2d(2),
                EqualizedConv2d(n_features, n_features * 2, 3, 1, 1),
                nn.LeakyReLU(0.2),
            )
        ])
        
        self.out = EqualizedLinear(n_features * 16, 1)
    
    def forward(self, x, alpha, progress):
        # Apply from RGB for the current resolution
        if progress == 0:
            x = self.from_rgb_layers[-1](x)
        else:
            # Fade in between resolutions
            x_curr = F.avg_pool2d(x, 2)
            x_curr = self.from_rgb_layers[len(self.from_rgb_layers) - progress](x_curr)
            x_prev = self.from_rgb_layers[len(self.from_rgb_layers) - progress + 1](x)
            x = alpha * x_curr + (1 - alpha) * x_prev
        
        # Apply progressive blocks
        for i in range(progress - 1, -1, -1):
            x = self.progressive_blocks[i](x)
        
        x = x.view(x.size(0), -1)
        x = self.out(x)
        return x

class ProGAN:
    def __init__(self, z_dim=512, n_features=16, img_channels=3, lr=0.001):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = Generator(z_dim, n_features, img_channels).to(self.device)
        self.discriminator = Discriminator(n_features, img_channels).to(self.device)
        
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0, 0.99))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0, 0.99))
        
        self.criterion = nn.BCELoss()
        self.z_dim = z_dim
    
    def train_step(self, real_data, alpha, progress):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data, alpha, progress)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise, alpha, progress)
        fake_output = self.discriminator(fake_data.detach(), alpha, progress)
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data, alpha, progress)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# Example usage
progan = ProGAN(z_dim=512, n_features=16, img_channels=3, lr=0.001)

# Sample training data for 64x64 images
sample_real_data = torch.randn(4, 3, 64, 64)

# Training at specific resolution (progress=3 for 64x64)
d_loss, g_loss = progan.train_step(sample_real_data, alpha=1.0, progress=3)

print(f"ProGAN Discriminator loss: {d_loss:.4f}")
print(f"ProGAN Generator loss: {g_loss:.4f}")

# Generate samples
with torch.no_grad():
    noise = torch.randn(5, 512, device=progan.device)
    generated = progan.generator(noise, alpha=1.0, progress=3)
    print(f"ProGAN generated samples shape: {generated.shape}")
```

## 8. How do you implement a StyleGAN architecture?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MappingNetwork(nn.Module):
    def __init__(self, z_dim, w_dim, n_mapping=8):
        super(MappingNetwork, self).__init__()
        self.z_dim = z_dim
        self.w_dim = w_dim
        
        # Mapping layers
        layers = []
        for i in range(n_mapping):
            in_dim = z_dim if i == 0 else w_dim
            layers.append(EqualizedLinear(in_dim, w_dim))
            layers.append(nn.LeakyReLU(0.2))
        
        self.mapping = nn.Sequential(*layers)
        self.norm = PixelNorm()
    
    def forward(self, z):
        # Apply pixel norm first
        z = self.norm(z)
        # Apply mapping network
        w = self.mapping(z)
        return w

class AdaIN(nn.Module):
    def __init__(self, w_dim, in_dim):
        super(AdaIN, self).__init__()
        self.w_dim = w_dim
        self.in_dim = in_dim
        
        # Style modulation
        self.style_mod = EqualizedLinear(w_dim, in_dim * 2)
        self.style_mod.linear.bias.data[:in_dim] = 1  # Initialize bias for scale to 1
        self.style_mod.linear.bias.data[in_dim:] = 0  # Initialize bias for bias to 0
    
    def forward(self, x, w):
        # Modulate with style
        style = self.style_mod(w).unsqueeze(2).unsqueeze(3)
        gamma, beta = style[:, :self.in_dim], style[:, self.in_dim:]
        out = gamma * F.instance_norm(x) + beta
        return out

class NoiseInjection(nn.Module):
    def __init__(self):
        super(NoiseInjection, self).__init__()
        self.weight = nn.Parameter(torch.zeros(1))
    
    def forward(self, x):
        batch_size, _, height, width = x.shape
        noise = torch.randn(batch_size, 1, height, width, device=x.device)
        return x + self.weight * noise

class StyledConvBlock(nn.Module):
    def __init__(self, in_dim, out_dim, w_dim, upsample=False):
        super(StyledConvBlock, self).__init__()
        self.upsample = upsample
        
        if upsample:
            self.upsample_layer = nn.Upsample(scale_factor=2, mode='bilinear', align_corners=True)
        
        # Convolution layers
        self.conv1 = EqualizedConv2d(in_dim, out_dim, 3, 1, 1)
        self.conv2 = EqualizedConv2d(out_dim, out_dim, 3, 1, 1)
        
        # AdaIN layers
        self.adain1 = AdaIN(w_dim, out_dim)
        self.adain2 = AdaIN(w_dim, out_dim)
        
        # Noise injection layers
        self.noise_injection1 = NoiseInjection()
        self.noise_injection2 = NoiseInjection()
        
        self.activation = nn.LeakyReLU(0.2)
    
    def forward(self, x, w):
        if self.upsample:
            x = self.upsample_layer(x)
        
        # First conv block
        x = self.conv1(x)
        x = self.noise_injection1(x)
        x = self.adain1(x, w)
        x = self.activation(x)
        
        # Second conv block
        x = self.conv2(x)
        x = self.noise_injection2(x)
        x = self.adain2(x, w)
        x = self.activation(x)
        
        return x

class ToRGB(nn.Module):
    def __init__(self, in_dim, w_dim):
        super(ToRGB, self).__init__()
        self.conv = EqualizedConv2d(in_dim, 3, 1, 1, 0)
        self.adain = AdaIN(w_dim, in_dim)
        self.bias = nn.Parameter(torch.zeros(1, 3, 1, 1))
    
    def forward(self, x, w):
        x = self.adain(x, w)
        x = self.conv(x)
        x = x + self.bias
        return x

class StyleGANGenerator(nn.Module):
    def __init__(self, z_dim, w_dim, n_features, img_channels=3, n_layers=8):
        super(StyleGANGenerator, self).__init__()
        self.z_dim = z_dim
        self.w_dim = w_dim
        self.n_features = n_features
        
        # Mapping network
        self.mapping = MappingNetwork(z_dim, w_dim)
        
        # Initial constant input: 4x4
        self.constant_input = nn.Parameter(torch.randn(1, n_features * 16, 4, 4))
        
        # First block
        self.initial_block = StyledConvBlock(n_features * 16, n_features * 16, w_dim)
        self.initial_to_rgb = ToRGB(n_features * 16, w_dim)
        
        # Progressive blocks
        in_dims = [n_features * 16, n_features * 16, n_features * 8, n_features * 4, n_features * 2]
        out_dims = [n_features * 16, n_features * 8, n_features * 4, n_features * 2, n_features]
        
        self.progressive_blocks = nn.ModuleList()
        self.to_rgb_layers = nn.ModuleList()
        
        for in_dim, out_dim in zip(in_dims, out_dims):
            self.progressive_blocks.append(StyledConvBlock(in_dim, out_dim, w_dim, upsample=True))
            self.to_rgb_layers.append(ToRGB(out_dim, w_dim))
    
    def forward(self, z, alpha=1.0, progress=0):
        # Get style vector
        w = self.mapping(z)
        
        # Start with constant input
        x = self.constant_input.expand(z.size(0), -1, -1, -1)
        
        # Apply initial block
        x = self.initial_block(x, w)
        out = self.initial_to_rgb(x, w)
        
        if progress == 0:
            return out
        
        # Apply progressive blocks
        for i in range(progress):
            x = self.progressive_blocks[i](x, w)
            skip_rgb = F.interpolate(out, scale_factor=2, mode='bilinear', align_corners=False)
            current_rgb = self.to_rgb_layers[i](x, w)
            out = alpha * current_rgb + (1 - alpha) * skip_rgb
        
        return out

class StyleGANDiscriminator(nn.Module):
    def __init__(self, img_channels=3, n_features=16):
        super(StyleGANDiscriminator, self).__init__()
        
        # Progressive blocks (reversed from generator)
        in_dims = [n_features, n_features * 2, n_features * 4, n_features * 8, n_features * 16]
        out_dims = [n_features * 2, n_features * 4, n_features * 8, n_features * 16, n_features * 16]
        
        self.from_rgb_layers = nn.ModuleList([
            nn.Conv2d(img_channels, n_features, 1),
            nn.Conv2d(img_channels, n_features * 2, 1),
            nn.Conv2d(img_channels, n_features * 4, 1),
            nn.Conv2d(img_channels, n_features * 8, 1),
            nn.Conv2d(img_channels, n_features * 16, 1),
        ])
        
        self.progressive_blocks = nn.ModuleList([
            nn.Sequential(
                EqualizedConv2d(in_dim, out_dim, 3, 1, 1),
                nn.LeakyReLU(0.2),
                nn.AvgPool2d(2)
            )
            for in_dim, out_dim in zip(in_dims, out_dims)
        ])
        
        # Final layers
        self.final_block = nn.Sequential(
            EqualizedConv2d(n_features * 16, n_features * 16, 4, 1, 0),
            nn.LeakyReLU(0.2),
        )
        
        self.out = EqualizedLinear(n_features * 16, 1)
    
    def forward(self, x, alpha=1.0, progress=0):
        # Apply from RGB for current resolution
        if progress == 0:
            x = self.from_rgb_layers[-1](x)
        else:
            # Fade in
            x_prev = F.avg_pool2d(x, 2)
            x_prev = self.from_rgb_layers[-progress](x_prev)
            x_curr = self.from_rgb_layers[-progress + 1](x)
            x = alpha * x_curr + (1 - alpha) * x_prev
        
        # Apply progressive blocks
        for i in range(progress, -1, -1):
            x = self.progressive_blocks[i](x)
        
        # Final layers
        x = self.final_block(x)
        x = x.view(x.size(0), -1)
        x = self.out(x)
        
        return x

class StyleGAN:
    def __init__(self, z_dim=512, w_dim=512, n_features=16, img_channels=3, lr=0.002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = StyleGANGenerator(z_dim, w_dim, n_features, img_channels).to(self.device)
        self.discriminator = StyleGANDiscriminator(img_channels, n_features).to(self.device)
        
        # Use different learning rates for different parts
        self.optimizer_G = torch.optim.Adam(
            self.generator.parameters(), lr=lr, betas=(0, 0.99)
        )
        self.optimizer_D = torch.optim.Adam(
            self.discriminator.parameters(), lr=lr, betas=(0, 0.99)
        )
        
        self.criterion = nn.BCELoss()
        self.z_dim = z_dim
    
    def train_step(self, real_data, alpha=1.0, progress=0):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data, alpha, progress)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise, alpha, progress)
        fake_output = self.discriminator(fake_data.detach(), alpha, progress)
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data, alpha, progress)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# Example usage
stylegan = StyleGAN(z_dim=512, w_dim=512, n_features=16, img_channels=3, lr=0.002)

# Sample training data
sample_real_data = torch.randn(4, 3, 64, 64)

# Training step
d_loss, g_loss = stylegan.train_step(sample_real_data, alpha=1.0, progress=3)

print(f"StyleGAN Discriminator loss: {d_loss:.4f}")
print(f"StyleGAN Generator loss: {g_loss:.4f}")

# Generate samples
with torch.no_grad():
    noise = torch.randn(5, 512, device=stylegan.device)
    generated = stylegan.generator(noise, alpha=1.0, progress=3)
    print(f"StyleGAN generated samples shape: {generated.shape}")
```

## 9. How do you implement a Diffusion Model?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

class SinusoidalPositionEmbeddings(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.dim = dim

    def forward(self, time):
        device = time.device
        half_dim = self.dim // 2
        embeddings = math.log(10000) / (half_dim - 1)
        embeddings = torch.exp(torch.arange(half_dim, device=device) * -embeddings)
        embeddings = time[:, None] * embeddings[None, :]
        embeddings = torch.cat((embeddings.sin(), embeddings.cos()), dim=-1)
        return embeddings

class Block(nn.Module):
    def __init__(self, in_ch, out_ch, time_emb_dim, up=False):
        super().__init__()
        self.time_mlp =  nn.Linear(time_emb_dim, out_ch)
        if up:
            self.conv1 = nn.Conv2d(2*in_ch, out_ch, 3, padding=1)
            self.transform = nn.ConvTranspose2d(out_ch, out_ch, 4, 2, 1)
        else:
            self.conv1 = nn.Conv2d(in_ch, out_ch, 3, padding=1)
            self.transform = nn.Conv2d(out_ch, out_ch, 4, 2, 1)
        self.conv2 = nn.Conv2d(out_ch, out_ch, 3, padding=1)
        self.bnorm1 = nn.BatchNorm2d(out_ch)
        self.bnorm2 = nn.BatchNorm2d(out_ch)
        self.relu  = nn.ReLU()
    
    def forward(self, x, t):
        # First Conv
        h = self.bnorm1(self.relu(self.conv1(x)))
        # Time embedding
        time_emb = self.relu(self.time_mlp(t))
        time_emb = time_emb[(..., ) + (None, ) * 2]
        h = h + time_emb
        # Second Conv
        h = self.bnorm2(self.relu(self.conv2(h)))
        # Down or Upsample
        return self.transform(h)

class SinusoidalPositionEmbeddings(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.dim = dim

    def forward(self, time):
        device = time.device
        half_dim = self.dim // 2
        embeddings = math.log(10000) / (half_dim - 1)
        embeddings = torch.exp(torch.arange(half_dim, device=device) * -embeddings)
        embeddings = time[:, None] * embeddings[None, :]
        embeddings = torch.cat((embeddings.sin(), embeddings.cos()), dim=-1)
        return embeddings

class SimpleUnet(nn.Module):
    """
    A simplified variant of the U-Net architecture for diffusion models.
    """
    def __init__(self):
        super(SimpleUnet, self).__init__()
        image_channels = 3
        down_channels = (64, 128, 256, 512, 1024)
        up_channels = (1024, 512, 256, 128, 64)
        out_dim = 3 
        time_emb_dim = 32

        # Time embedding
        self.time_mlp = nn.Sequential(
                SinusoidalPositionEmbeddings(time_emb_dim),
                nn.Linear(time_emb_dim, time_emb_dim),
                nn.ReLU()
            )
        
        # Initial projection
        self.conv0 = nn.Conv2d(image_channels, down_channels[0], 3, padding=1)

        # Downsample
        self.downs = nn.ModuleList([Block(down_channels[i], down_channels[i+1], time_emb_dim) 
                                    for i in range(len(down_channels)-1)])
        # Upsample
        self.ups = nn.ModuleList([Block(up_channels[i], up_channels[i+1], time_emb_dim, up=True) 
                                  for i in range(len(up_channels)-1)])

        # Edit: Corrected the final layer to match the up-sampling
        self.output = nn.Conv2d(up_channels[-1], out_dim, 1)

    def forward(self, x, timestep):
        # Embedd time
        t = self.time_mlp(timestep)
        # Initial conv
        x = self.conv0(x)
        # Unet
        residual_inputs = []
        for down in self.downs:
            x = down(x, t)
            # Store residual
            residual_inputs.append(x)
        for up in self.ups:
            # Get the last residual
            residual_x = residual_inputs.pop()
            # Add residual x as additional channels
            x = torch.cat((x, residual_x), dim=1) 
            x = up(x, t)
        return self.output(x)

def linear_beta_schedule(timesteps, start=0.0001, end=0.02):
    return torch.linspace(start, end, timesteps)

def get_index_from_list(vals, t, x_shape):
    """
    Returns a specific index t of a passed list of values vals
    while considering the batch dimension.
    """
    batch_size = t.shape[0]
    out = vals.gather(-1, t.cpu())
    return out.reshape(batch_size, *((1,) * (len(x_shape) - 1))).to(t.device)

def forward_diffusion_sample(x_0, t, device="cpu"):
    """
    Takes an image and a timestep as input and 
    returns the noisy version of it
    """
    noise = torch.randn_like(x_0)
    sqrt_alphas_cumprod_t = get_index_from_list(sqrt_alphas_cumprod, t, x_0.shape)
    sqrt_one_minus_alphas_cumprod_t = get_index_from_list(
        sqrt_one_minus_alphas_cumprod, t, x_0.shape
    )
    # Mean + variance
    return sqrt_alphas_cumprod_t.to(device) * x_0.to(device) \
    + sqrt_one_minus_alphas_cumprod_t.to(device) * noise.to(device)

T = 300
betas = linear_beta_schedule(timesteps=T)

# Pre-calculate different terms for closed form
alphas = 1. - betas
alphas_cumprod = torch.cumprod(alphas, axis=0)
alphas_cumprod_prev = F.pad(alphas_cumprod[:-1], (1, 0), value=1.0)
sqrt_recip_alphas = torch.sqrt(1.0 / alphas)
sqrt_alphas_cumprod = torch.sqrt(alphas_cumprod)
sqrt_one_minus_alphas_cumprod = torch.sqrt(1. - alphas_cumprod)
posterior_variance = betas * (1. - alphas_cumprod_prev) / (1. - alphas_cumprod)

class DiffusionModel:
    def __init__(self, T=300, device='cuda' if torch.cuda.is_available() else 'cpu'):
        self.device = device
        self.T = T
        
        # Create model
        self.model = SimpleUnet().to(device)
        
        # Set up optimizer
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=0.001)
        
        # Set up diffusion parameters
        self.betas = linear_beta_schedule(timesteps=T).to(device)
        self.alphas = 1. - self.betas
        self.alphas_cumprod = torch.cumprod(self.alphas, axis=0)
        self.alphas_cumprod_prev = F.pad(self.alphas_cumprod[:-1], (1, 0), value=1.0)
        
        self.sqrt_recip_alphas = torch.sqrt(1.0 / self.alphas)
        self.sqrt_alphas_cumprod = torch.sqrt(self.alphas_cumprod)
        self.sqrt_one_minus_alphas_cumprod = torch.sqrt(1. - self.alphas_cumprod)
        self.posterior_variance = self.betas * (1. - self.alphas_cumprod_prev) / (1. - self.alphas_cumprod)
    
    def get_index_from_list(self, vals, t, x_shape):
        batch_size = t.shape[0]
        out = vals.gather(-1, t.cpu())
        return out.reshape(batch_size, *((1,) * (len(x_shape) - 1))).to(t.device)
    
    def forward_diffusion_sample(self, x_0, t):
        noise = torch.randn_like(x_0)
        sqrt_alphas_cumprod_t = self.get_index_from_list(self.sqrt_alphas_cumprod, t, x_0.shape)
        sqrt_one_minus_alphas_cumprod_t = self.get_index_from_list(
            self.sqrt_one_minus_alphas_cumprod, t, x_0.shape
        )
        return sqrt_alphas_cumprod_t.to(self.device) * x_0.to(self.device) \
               + sqrt_one_minus_alphas_cumprod_t.to(self.device) * noise.to(self.device)
    
    def train_step(self, x_0):
        self.model.train()
        self.optimizer.zero_grad()
        
        # Sample random timesteps
        t = torch.randint(0, self.T, (x_0.shape[0],), device=self.device).long()
        
        # Add noise to images
        x_noisy = self.forward_diffusion_sample(x_0, t)
        
        # Predict noise
        predicted_noise = self.model(x_noisy, t)
        
        # Calculate loss
        loss = F.mse_loss(predicted_noise, torch.randn_like(x_0))
        
        # Backward pass
        loss.backward()
        self.optimizer.step()
        
        return loss.item()
    
    @torch.no_grad()
    def sample_timestep(self, x, t):
        """
        Calls the model to predict the noise in the image and returns 
        the denoised image. 
        Applies noise to this image, if we are not in the last step yet.
        """
        betas_t = self.get_index_from_list(self.betas, t, x.shape)
        sqrt_one_minus_alphas_cumprod_t = self.get_index_from_list(
            self.sqrt_one_minus_alphas_cumprod, t, x.shape
        )
        sqrt_recip_alphas_t = self.get_index_from_list(self.sqrt_recip_alphas, t, x.shape)
        
        # Call model (current image - noise prediction)
        model_mean = sqrt_recip_alphas_t * (
            x - betas_t * self.model(x, t) / sqrt_one_minus_alphas_cumprod_t
        )
        posterior_variance_t = self.get_index_from_list(self.posterior_variance, t, x.shape)
        
        if t == 0:
            return model_mean
        else:
            noise = torch.randn_like(x)
            return model_mean + torch.sqrt(posterior_variance_t) * noise 

    @torch.no_grad()
    def sample(self, size, img_channels=3):
        """Samples images from the model"""
        self.model.eval()
        img = torch.randn((size, img_channels, 64, 64), device=self.device)
        imgs = []
        
        for i in range(0, self.T)[::-1]:
            t = torch.full((size,), i, device=self.device, dtype=torch.long)
            img = self.sample_timestep(img, t)
            imgs.append(img.cpu().numpy())
        
        return imgs

# Example usage
diffusion_model = DiffusionModel(T=300)

# Sample training step
sample_images = torch.randn(4, 3, 64, 64).to(diffusion_model.device)
loss = diffusion_model.train_step(sample_images)

print(f"Diffusion model loss: {loss:.4f}")

# Generate samples (this would take many more training steps to produce good results)
with torch.no_grad():
    samples = diffusion_model.sample(2)
    print(f"Generated samples shape: {len(samples)} timesteps, each of shape {samples[0].shape}")
```

## 10. How do you implement a Normalizing Flow?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MADE(nn.Module):
    """Masked Autoencoder for Distribution Estimation"""
    def __init__(self, input_size, hidden_size, n_hidden=1):
        super(MADE, self).__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.n_hidden = n_hidden
        
        # Create masks for autoregressive property
        self.masks = self._create_masks()
        
        # Define layers
        layers = []
        sizes = [input_size] + [hidden_size] * n_hidden + [input_size * 2]  # mean and scale
        for i in range(len(sizes) - 1):
            layer = nn.Linear(sizes[i], sizes[i+1])
            layer.register_forward_hook(self._set_mask(i))
            layers.append(layer)
        
        self.layers = nn.ModuleList(layers)
        
    def _create_masks(self):
        masks = []
        d = self.input_size
        h = self.hidden_size
        
        # Input to hidden
        M0 = torch.zeros(h, d)
        for i in range(h):
            M0[i, :torch.randint(1, d+1, (1,)).item()] = 1
        
        # Hidden to hidden
        Mh = torch.zeros(h, h)
        for i in range(h):
            Mh[i, :torch.randint(1, h+1, (1,)).item()] = 1
        
        # Hidden to output (ensure each output only depends on inputs < its position)
        Mout = torch.zeros(d, h)
        for i in range(d):
            Mout[i, torch.randint(i, h if i > 0 else h+1, (1,)).item():] = 1
        
        masks.extend([M0, Mh] * (self.n_hidden) + [Mout])
        return masks
    
    def _set_mask(self, layer_idx):
        def hook(module, input, output):
            if hasattr(self, 'masks') and layer_idx < len(self.masks):
                module.weight.data *= self.masks[layer_idx]
            return output
        return hook
    
    def forward(self, x):
        h = x
        for layer in self.layers[:-1]:
            h = F.relu(layer(h))
        
        # Output mean and log scale
        out = self.layers[-1](h)
        mean, log_scale = torch.chunk(out, 2, dim=-1)
        log_scale = torch.tanh(log_scale)  # Keep log scale bounded
        return mean, log_scale

class RealNVP(nn.Module):
    """Real NVP (Non-volume preserving) flow"""
    def __init__(self, input_size, hidden_size, n_coupling_layers=4):
        super(RealNVP, self).__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.n_coupling_layers = n_coupling_layers
        
        # Create coupling layers
        self.coupling_layers = nn.ModuleList()
        for i in range(n_coupling_layers):
            # Use alternating masks
            mask = torch.ones(input_size)
            mask[::2] = 0 if i % 2 == 0 else 1
            self.coupling_layers.append(
                CouplingLayer(input_size, hidden_size, mask)
            )
    
    def forward(self, x):
        log_det = torch.zeros(x.size(0), device=x.device)
        out = x
        
        for layer in self.coupling_layers:
            out, log_det_i = layer(out)
            log_det += log_det_i
        
        return out, log_det
    
    def inverse(self, z):
        log_det = torch.zeros(z.size(0), device=z.device)
        out = z
        
        for layer in reversed(self.coupling_layers):
            out, log_det_i = layer.inverse(out)
            log_det += log_det_i
        
        return out, log_det

class CouplingLayer(nn.Module):
    def __init__(self, input_size, hidden_size, mask):
        super(CouplingLayer, self).__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.register_buffer('mask', mask)
        
        # Neural networks for translation and scaling
        self.scale_net = MADE(input_size, hidden_size, n_hidden=2)
        self.translate_net = MADE(input_size, hidden_size, n_hidden=2)
    
    def forward(self, x):
        # Split input based on mask
        x1 = x * self.mask
        x2 = x * (1 - self.mask)
        
        # Compute scaling and translation for masked part
        scale_mean, scale_log = self.scale_net(x1)
        translate_mean, _ = self.translate_net(x1)
        
        # Apply transformation
        y2 = x2 * torch.exp(scale_log) + translate_mean
        y = x1 + y2 * (1 - self.mask)
        
        # Log determinant of Jacobian
        log_det = torch.sum(scale_log * (1 - self.mask), dim=1)
        
        return y, log_det
    
    def inverse(self, y):
        # Split input based on mask
        y1 = y * self.mask
        y2 = y * (1 - self.mask)
        
        # Compute scaling and translation for masked part
        scale_mean, scale_log = self.scale_net(y1)
        translate_mean, _ = self.translate_net(y1)
        
        # Apply inverse transformation
        x2 = (y2 - translate_mean) * torch.exp(-scale_log)
        x = y1 + x2 * (1 - self.mask)
        
        # Log determinant of Jacobian (negative for inverse)
        log_det = -torch.sum(scale_log * (1 - self.mask), dim=1)
        
        return x, log_det

class NormalizingFlow(nn.Module):
    def __init__(self, input_size, hidden_size, n_flows=4):
        super(NormalizingFlow, self).__init__()
        self.input_size = input_size
        
        # Base distribution (standard normal)
        self.register_buffer('base_mean', torch.zeros(input_size))
        self.register_buffer('base_cov', torch.eye(input_size))
        
        # Flow layers
        self.flows = nn.ModuleList([
            RealNVP(input_size, hidden_size, n_coupling_layers=2) 
            for _ in range(n_flows)
        ])
    
    def forward(self, x):
        log_det = torch.zeros(x.size(0), device=x.device)
        z = x
        
        for flow in self.flows:
            z, log_det_i = flow(z)
            log_det += log_det_i
        
        # Log probability in latent space (standard normal)
        log_prob_z = -0.5 * torch.sum((z - self.base_mean) ** 2, dim=1) - 0.5 * torch.log(2 * torch.pi) * self.input_size
        
        # Total log probability
        log_prob_x = log_prob_z + log_det
        return z, log_prob_x
    
    def inverse(self, z):
        x = z
        log_det = torch.zeros(z.size(0), device=z.device)
        
        for flow in reversed(self.flows):
            x, log_det_i = flow.inverse(x)
            log_det += log_det_i
        
        return x, log_det
    
    def sample(self, n_samples):
        # Sample from base distribution
        z = torch.randn(n_samples, self.input_size, device=self.base_mean.device)
        
        # Transform through inverse flows
        x, _ = self.inverse(z)
        return x
    
    def log_prob(self, x):
        _, log_prob = self.forward(x)
        return log_prob

# Example usage
input_size = 784  # For flattened 28x28 images
flow_model = NormalizingFlow(input_size=input_size, hidden_size=256, n_flows=4)

# Sample input
sample_input = torch.randn(32, input_size)

# Forward pass
z, log_prob = flow_model(sample_input)
print(f"Input shape: {sample_input.shape}")
print(f"Latent shape: {z.shape}")
print(f"Log probability shape: {log_prob.shape}")

# Sample from model
samples = flow_model.sample(5)
print(f"Sampled data shape: {samples.shape}")

# Calculate log probability
log_prob = flow_model.log_prob(sample_input)
print(f"Log probability: {log_prob.mean().item():.4f}")
```

## 11. How do you implement an Autoregressive Model?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PixelCNN(nn.Module):
    """PixelCNN for autoregressive image generation"""
    def __init__(self, input_channels=3, n_filters=128, n_layers=12, n_classes=256):
        super(PixelCNN, self).__init__()
        
        self.input_channels = input_channels
        self.n_filters = n_filters
        self.n_classes = n_classes
        
        # Input layer (with masking)
        self.input_conv = MaskedConv2d('A', input_channels, n_filters, 7, 1, 3)
        self.input_bn = nn.BatchNorm2d(n_filters)
        
        # Residual layers
        self.residual_layers = nn.ModuleList([
            ResidualBlock(n_filters, n_filters) for _ in range(n_layers)
        ])
        
        # Output layers
        self.output_convs = nn.Sequential(
            nn.Conv2d(n_filters, n_filters, 1),
            nn.ReLU(),
            nn.Conv2d(n_filters, input_channels * n_classes, 1)
        )
    
    def forward(self, x):
        # Initial convolution with masking
        x = self.input_conv(x)
        x = F.relu(self.input_bn(x))
        
        # Residual blocks
        for layer in self.residual_layers:
            x = layer(x)
        
        # Output convolution
        x = self.output_convs(x)
        
        # Reshape for output
        batch_size, _, height, width = x.shape
        x = x.view(batch_size, self.input_channels, self.n_classes, height, width)
        
        return x

class MaskedConv2d(nn.Conv2d):
    """Conv2d with masking for autoregressive property"""
    def __init__(self, mask_type, *args, **kwargs):
        super(MaskedConv2d, self).__init__(*args, **kwargs)
        self.register_buffer('mask', torch.zeros_like(self.weight))
        self.create_mask(mask_type)
    
    def create_mask(self, mask_type):
        # Get center of kernel
        center_h, center_w = self.kernel_size[0] // 2, self.kernel_size[1] // 2
        
        # Create mask
        self.mask.fill_(1)
        
        # Zero out pixels below center
        self.mask[:, :, center_h + 1:, :] = 0
        
        # For mask type A, also zero out center pixel
        if mask_type == 'A':
            self.mask[:, :, center_h, center_w + 1:] = 0
        elif mask_type == 'B':
            # Mask type B allows center pixel
            pass
    
    def forward(self, x):
        # Apply mask to weights
        self.weight.data *= self.mask
        return super(MaskedConv2d, self).forward(x)

class ResidualBlock(nn.Module):
    def __init__(self, n_channels):
        super(ResidualBlock, self).__init__()
        
        self.conv1 = MaskedConv2d('B', n_channels, n_channels // 2, 1)
        self.conv2 = MaskedConv2d('B', n_channels // 2, n_channels // 2, 3, 1, 1)
        self.conv3 = MaskedConv2d('B', n_channels // 2, n_channels, 1)
        
        self.bn1 = nn.BatchNorm2d(n_channels // 2)
        self.bn2 = nn.BatchNorm2d(n_channels // 2)
        self.bn3 = nn.BatchNorm2d(n_channels)
    
    def forward(self, x):
        residual = x
        
        out = F.relu(self.bn1(self.conv1(x)))
        out = F.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        
        return x + out

class WaveNet(nn.Module):
    """WaveNet for autoregressive audio generation"""
    def __init__(self, n_layers=10, n_channels=32, n_classes=256, n_conditioning=0):
        super(WaveNet, self).__init__()
        
        self.n_layers = n_layers
        self.n_channels = n_channels
        self.n_classes = n_classes
        self.n_conditioning = n_conditioning
        
        # Input embedding
        self.embed = nn.Embedding(n_classes, n_channels)
        
        # Dilated convolutions with different dilation rates
        self.dil_conv_layers = nn.ModuleList()
        self.res_conv_layers = nn.ModuleList()
        self.skip_conv_layers = nn.ModuleList()
        
        for i in range(n_layers):
            dilation = 2 ** i
            # Dilated convolution
            self.dil_conv_layers.append(
                nn.Conv1d(n_channels, 2 * n_channels, 2, dilation=dilation, padding=dilation)
            )
            # Residual and skip connections
            self.res_conv_layers.append(nn.Conv1d(n_channels, n_channels, 1))
            self.skip_conv_layers.append(nn.Conv1d(n_channels, n_channels, 1))
        
        # Final layers
        self.final_layers = nn.Sequential(
            nn.ReLU(),
            nn.Conv1d(n_channels, n_channels, 1),
            nn.ReLU(),
            nn.Conv1d(n_channels, n_classes, 1)
        )
    
    def forward(self, x, conditioning=None):
        # Input embedding
        x = self.embed(x).transpose(1, 2)  # [B, C, T]
        
        skip_connections = []
        
        for i in range(self.n_layers):
            # Dilated convolution
            dil_out = self.dil_conv_layers[i](x)
            
            # Split for gated activation
            tanh_out = torch.tanh(dil_out[:, :self.n_channels, :])
            sigm_out = torch.sigmoid(dil_out[:, self.n_channels:, :])
            gated_out = tanh_out * sigm_out
            
            # Residual connection
            res_out = self.res_conv_layers[i](gated_out)
            x = x + res_out
            
            # Skip connection
            skip_out = self.skip_conv_layers[i](gated_out)
            skip_connections.append(skip_out)
        
        # Sum all skip connections
        out = sum(skip_connections)
        
        # Final layers
        out = self.final_layers(out)
        
        return out.transpose(1, 2)  # [B, T, n_classes]

class AutoregressiveModel:
    def __init__(self, model_type='pixelcnn', **kwargs):
        self.model_type = model_type
        
        if model_type == 'pixelcnn':
            self.model = PixelCNN(**kwargs)
        elif model_type == 'wavenet':
            self.model = WaveNet(**kwargs)
        else:
            raise ValueError(f"Unknown model type: {model_type}")
        
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=0.001)
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, x):
        self.model.train()
        self.optimizer.zero_grad()
        
        # Forward pass
        if self.model_type == 'pixelcnn':
            logits = self.model(x)
            # Reshape for loss calculation
            batch_size, channels, classes, height, width = logits.shape
            logits = logits.permute(0, 3, 4, 1, 2).contiguous().view(-1, classes)
            targets = x.permute(0, 2, 3, 1).contiguous().view(-1).long()
        else:  # wavenet
            targets = x[:, 1:].contiguous().view(-1).long()  # Shift by one for autoregressive
            logits = self.model(x[:, :-1])  # All but last for input
            logits = logits.contiguous().view(-1, logits.size(-1))
        
        # Calculate loss
        loss = self.criterion(logits, targets)
        
        # Backward pass
        loss.backward()
        self.optimizer.step()
        
        return loss.item()
    
    @torch.no_grad()
    def generate(self, shape, temperature=1.0):
        self.model.eval()
        
        if self.model_type == 'pixelcnn':
            # Generate image pixel by pixel
            batch_size = shape[0]
            channels, height, width = shape[1], shape[2], shape[3]
            generated = torch.zeros(batch_size, channels, height, width, device=self.device)
            
            for i in range(height):
                for j in range(width):
                    for c in range(channels):
                        logits = self.model(generated)
                        probs = F.softmax(logits[:, c, :, i, j] / temperature, dim=-1)
                        generated[:, c, i, j] = torch.multinomial(probs, 1).float()
        
        elif self.model_type == 'wavenet':
            # Generate sequence step by step
            batch_size, seq_len = shape[0], shape[1]
            generated = torch.zeros(batch_size, seq_len, device=self.device, dtype=torch.long)
            
            for i in range(seq_len):
                if i == 0:
                    # Start with first token (could be special start token)
                    logits = self.model(generated[:, :i+1])
                else:
                    logits = self.model(generated[:, :i])
                
                probs = F.softmax(logits[:, -1, :] / temperature, dim=-1)
                generated[:, i] = torch.multinomial(probs, 1).squeeze(-1)
        
        return generated

# Example usage
# PixelCNN example
pixelcnn = AutoregressiveModel(
    model_type='pixelcnn',
    input_channels=1,  # For grayscale images
    n_filters=64,
    n_layers=4,
    n_classes=256  # For 8-bit images
)

# Sample training
sample_images = torch.randint(0, 256, (4, 1, 28, 28)).float() / 255.0
loss = pixelcnn.train_step(sample_images)
print(f"PixelCNN loss: {loss:.4f}")

# Generate sample
generated = pixelcnn.generate((2, 1, 28, 28))
print(f"PixelCNN generated shape: {generated.shape}")

# WaveNet example
wavenet = AutoregressiveModel(
    model_type='wavenet',
    n_layers=4,
    n_channels=32,
    n_classes=256
)

# Sample training for audio-like data
sample_audio = torch.randint(0, 256, (4, 100))  # Sequences of length 100
loss = wavenet.train_step(sample_audio)
print(f"WaveNet loss: {loss:.4f}")

# Generate sample
generated_audio = wavenet.generate((2, 50))
print(f"WaveNet generated shape: {generated_audio.shape}")
```

## 12. How do you implement a Variational Autoencoder with Normalizing Flows?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.distributions import Normal, Uniform

class PlanarFlow(nn.Module):
    """Planar Flow transformation"""
    def __init__(self, dim):
        super(PlanarFlow, self).__init__()
        self.u = nn.Parameter(torch.randn(1, dim))
        self.w = nn.Parameter(torch.randn(1, dim))
        self.b = nn.Parameter(torch.randn(1))
    
    def forward(self, z):
        # Ensure invertibility
        wu = torch.sum(self.w * self.u)
        self.u.data = self.u.data - (wu - torch.log(1 + torch.exp(wu))) * self.w / torch.sum(self.w ** 2)
        
        activation = torch.sum(self.w * z, dim=1, keepdim=True) + self.b
        z_out = z + self.u * torch.tanh(activation)
        
        # Compute log determinant
        psi = (1 - torch.tanh(activation) ** 2) * self.w
        log_det = torch.log(torch.abs(1 + torch.sum(psi * self.u, dim=1)) + 1e-8)
        
        return z_out, log_det

class RadialFlow(nn.Module):
    """Radial Flow transformation"""
    def __init__(self, dim):
        super(RadialFlow, self).__init__()
        self.dim = dim
        self.z0 = nn.Parameter(torch.randn(1, dim))
        self.alpha = nn.Parameter(torch.randn(1))
        self.beta = nn.Parameter(torch.randn(1))
    
    def forward(self, z):
        # Ensure invertibility
        alpha_hat = F.softplus(self.alpha)
        beta_hat = -alpha_hat + F.softplus(self.beta)
        
        diff = z - self.z0
        r = torch.norm(diff, dim=1, keepdim=True)
        h = 1 / (alpha_hat + r)
        z_out = z + beta_hat * h * diff
        
        # Compute log determinant
        h_prime = -1 / (alpha_hat + r)**2
        log_det = torch.log(torch.abs(1 + beta_hat * h + beta_hat * h_prime * r) + 1e-8) * self.dim
        
        return z_out, log_det

class VAEWithFlows(nn.Module):
    def __init__(self, input_dim, hidden_dim, latent_dim, n_flows=4):
        super(VAEWithFlows, self).__init__()
        
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.latent_dim = latent_dim
        self.n_flows = n_flows
        
        # Encoder
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        self.fc_mu = nn.Linear(hidden_dim, latent_dim)
        self.fc_logvar = nn.Linear(hidden_dim, latent_dim)
        
        # Decoder
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, input_dim),
            nn.Sigmoid()
        )
        
        # Normalizing flows
        flows = []
        for i in range(n_flows):
            # Alternate between planar and radial flows
            if i % 2 == 0:
                flows.append(PlanarFlow(latent_dim))
            else:
                flows.append(RadialFlow(latent_dim))
        self.flows = nn.ModuleList(flows)
        
        # Learnable parameters for initial distribution
        self.prior_mean = nn.Parameter(torch.zeros(latent_dim))
        self.prior_logvar = nn.Parameter(torch.zeros(latent_dim))
    
    def encode(self, x):
        h = self.encoder(x)
        mu = self.fc_mu(h)
        logvar = self.fc_logvar(h)
        return mu, logvar
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def forward(self, x):
        # Encode
        mu, logvar = self.encode(x)
        z_0 = self.reparameterize(mu, logvar)
        
        # Apply normalizing flows
        z_k = z_0
        sum_log_det = torch.zeros(x.size(0), device=x.device)
        
        for flow in self.flows:
            z_k, log_det = flow(z_k)
            sum_log_det += log_det
        
        # Decode
        recon_x = self.decoder(z_k)
        
        # Compute prior log probability
        prior = Normal(self.prior_mean, torch.exp(0.5 * self.prior_logvar))
        log_p_z = torch.sum(prior.log_prob(z_k), dim=1)
        
        # Compute approximate posterior log probability
        log_q_z0 = torch.sum(Normal(mu, torch.exp(0.5 * logvar)).log_prob(z_0), dim=1)
        
        return recon_x, z_k, mu, logvar, log_p_z, log_q_z0, sum_log_det
    
    def encode_and_transform(self, x):
        """Encode and apply flow transformations"""
        mu, logvar = self.encode(x)
        z_0 = self.reparameterize(mu, logvar)
        
        z_k = z_0
        sum_log_det = torch.zeros(x.size(0), device=x.device)
        
        for flow in self.flows:
            z_k, log_det = flow(z_k)
            sum_log_det += log_det
        
        return z_k, sum_log_det
    
    def decode_from_latent(self, z):
        """Decode from latent space"""
        return self.decoder(z)

def vae_with_flows_loss(recon_x, x, log_p_z, log_q_z0, sum_log_det, beta=1.0):
    # Reconstruction loss
    recon_loss = F.binary_cross_entropy(recon_x, x, reduction='sum') / x.size(0)
    
    # KL divergence with flows
    kl_loss = torch.mean(log_q_z0 - log_p_z - sum_log_det)
    
    return recon_loss + beta * kl_loss, recon_loss, kl_loss

# Example usage
input_dim = 784  # 28x28 images flattened
vae_flows = VAEWithFlows(input_dim=input_dim, hidden_dim=400, latent_dim=20, n_flows=4)

# Sample forward pass
sample_input = torch.randn(32, input_dim)
recon_x, z_k, mu, logvar, log_p_z, log_q_z0, sum_log_det = vae_flows(sample_input)

loss, recon_loss, kl_loss = vae_with_flows_loss(
    recon_x, sample_input, log_p_z, log_q_z0, sum_log_det
)

print(f"VAE with Flows Input shape: {sample_input.shape}")
print(f"Reconstructed shape: {recon_x.shape}")
print(f"Latent shape: {z_k.shape}")
print(f"Total loss: {loss.item():.4f}")
print(f"Reconstruction loss: {recon_loss.item():.4f}")
print(f"KL loss: {kl_loss.item():.4f}")

# Generate samples
with torch.no_grad():
    # Sample from prior
    z_prior = torch.randn(5, 20)
    generated = vae_flows.decode_from_latent(z_prior)
    print(f"Generated samples shape: {generated.shape}")
```

## 13. How do you implement a Wasserstein GAN (WGAN)?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim

class WGANGenerator(nn.Module):
    def __init__(self, noise_dim, img_dim, hidden_dim=128):
        super(WGANGenerator, self).__init__()
        
        self.model = nn.Sequential(
            nn.Linear(noise_dim, hidden_dim),
            nn.ReLU(True),
            nn.Linear(hidden_dim, hidden_dim * 2),
            nn.ReLU(True),
            nn.Linear(hidden_dim * 2, hidden_dim * 4),
            nn.ReLU(True),
            nn.Linear(hidden_dim * 4, img_dim),
            nn.Tanh()
        )
    
    def forward(self, x):
        return self.model(x)

class WGANCritic(nn.Module):
    def __init__(self, img_dim, hidden_dim=128):
        super(WGANCritic, self).__init__()
        
        self.model = nn.Sequential(
            nn.Linear(img_dim, hidden_dim),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim, hidden_dim * 2),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim * 2, hidden_dim * 4),
            nn.LeakyReLU(0.2),
            nn.Linear(hidden_dim * 4, 1)
        )
    
    def forward(self, x):
        return self.model(x)

class WGAN:
    def __init__(self, noise_dim, img_dim, lr=0.00005, clip_value=0.01, n_critic=5):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = WGANGenerator(noise_dim, img_dim).to(self.device)
        self.critic = WGANCritic(img_dim).to(self.device)
        
        self.optimizer_G = optim.RMSprop(self.generator.parameters(), lr=lr)
        self.optimizer_C = optim.RMSprop(self.critic.parameters(), lr=lr)
        
        self.noise_dim = noise_dim
        self.clip_value = clip_value
        self.n_critic = n_critic  # Train critic more often
    
    def train_step(self, real_data, step):
        batch_size = real_data.size(0)
        
        # Train Critic
        self.optimizer_C.zero_grad()
        
        # Real data
        real_output = self.critic(real_data)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.critic(fake_data.detach())
        
        # WGAN loss: maximize (real_output - fake_output)
        critic_loss = -(torch.mean(real_output) - torch.mean(fake_output))
        critic_loss.backward()
        self.optimizer_C.step()
        
        # Clip critic weights (Lipschitz constraint)
        for p in self.critic.parameters():
            p.data.clamp_(-self.clip_value, self.clip_value)
        
        # Train Generator every n_critic steps
        generator_loss = None
        if step % self.n_critic == 0:
            self.optimizer_G.zero_grad()
            
            # Generate fake data
            fake_data = self.generator(noise)
            fake_output = self.critic(fake_data)
            
            # Generator tries to maximize critic output (minimize -critic_output)
            generator_loss = -torch.mean(fake_output)
            generator_loss.backward()
            self.optimizer_G.step()
        
        return critic_loss.item(), generator_loss.item() if generator_loss else None

# Example usage
noise_dim = 100
img_dim = 784  # For 28x28 images
wgan = WGAN(noise_dim, img_dim, lr=0.00005, n_critic=5)

# Sample training step
sample_real_data = torch.randn(32, img_dim)
c_loss, g_loss = wgan.train_step(sample_real_data, step=0)

print(f"WGAN Critic loss: {c_loss:.4f}")
print(f"WGAN Generator loss: {g_loss:.4f}" if g_loss else "WGAN Generator not trained this step")

# Generate samples
with torch.no_grad():
    noise = torch.randn(5, noise_dim, device=wgan.device)
    generated = wgan.generator(noise)
    print(f"WGAN generated samples shape: {generated.shape}")

# WGAN-GP (Wasserstein GAN with Gradient Penalty)
class WGAN_GP:
    def __init__(self, noise_dim, img_dim, lr=0.0001, lambda_gp=10):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = WGANGenerator(noise_dim, img_dim).to(self.device)
        self.critic = WGANCritic(img_dim).to(self.device)
        
        self.optimizer_G = optim.Adam(self.generator.parameters(), lr=lr, betas=(0, 0.9))
        self.optimizer_C = optim.Adam(self.critic.parameters(), lr=lr, betas=(0, 0.9))
        
        self.noise_dim = noise_dim
        self.lambda_gp = lambda_gp
    
    def compute_gradient_penalty(self, real_data, fake_data):
        """Compute gradient penalty for WGAN-GP"""
        batch_size = real_data.size(0)
        
        # Sample random interpolation
        alpha = torch.rand(batch_size, 1, device=self.device)
        alpha = alpha.expand(real_data.size())
        
        interpolated = alpha * real_data + ((1 - alpha) * fake_data)
        interpolated = interpolated.requires_grad_(True)
        
        # Critic's output for interpolated data
        critic_interpolated = self.critic(interpolated)
        
        # Compute gradients
        gradients = torch.autograd.grad(
            outputs=critic_interpolated,
            inputs=interpolated,
            grad_outputs=torch.ones(interpolated.size(0), 1, device=self.device),
            create_graph=True,
            retain_graph=True,
            only_inputs=True
        )[0]
        
        # Compute gradient penalty
        gradients = gradients.view(gradients.size(0), -1)
        gradient_penalty = torch.mean((gradients.norm(2, dim=1) - 1) ** 2)
        
        return gradient_penalty
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        
        # Train Critic
        self.optimizer_C.zero_grad()
        
        # Real data
        real_output = self.critic(real_data)
        
        # Fake data
        noise = torch.randn(batch_size, self.noise_dim, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.critic(fake_data.detach())
        
        # WGAN loss with gradient penalty
        critic_loss = -(torch.mean(real_output) - torch.mean(fake_output))
        
        # Compute gradient penalty
        gradient_penalty = self.compute_gradient_penalty(real_data.data, fake_data.data)
        critic_loss += self.lambda_gp * gradient_penalty
        
        critic_loss.backward()
        self.optimizer_C.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.critic(fake_data)
        generator_loss = -torch.mean(fake_output)
        generator_loss.backward()
        self.optimizer_G.step()
        
        return critic_loss.item(), generator_loss.item()

# Example usage of WGAN-GP
wgan_gp = WGAN_GP(noise_dim=100, img_dim=784, lr=0.0001)
sample_real_data = torch.randn(32, 784)
c_loss_gp, g_loss_gp = wgan_gp.train_step(sample_real_data)

print(f"WGAN-GP Critic loss: {c_loss_gp:.4f}")
print(f"WGAN-GP Generator loss: {g_loss_gp:.4f}")
```

## 14. How do you implement a CycleGAN for image-to-image translation?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualBlock(nn.Module):
    def __init__(self, channels):
        super(ResidualBlock, self).__init__()
        self.conv1 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn1 = nn.InstanceNorm2d(channels)
        self.conv2 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn2 = nn.InstanceNorm2d(channels)

    def forward(self, x):
        residual = x
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out = out + residual  # Skip connection
        return out

class Generator(nn.Module):
    """Generator for CycleGAN (ResNet-based)"""
    def __init__(self, input_channels, output_channels, n_residual_blocks=9):
        super(Generator, self).__init__()
        
        # Initial convolution block
        model = [
            nn.Conv2d(input_channels, 64, 7, padding=3),
            nn.InstanceNorm2d(64),
            nn.ReLU(inplace=True)
        ]
        
        # Downsampling
        in_channels = 64
        out_channels = in_channels * 2
        for _ in range(2):
            model += [
                nn.Conv2d(in_channels, out_channels, 3, stride=2, padding=1),
                nn.InstanceNorm2d(out_channels),
                nn.ReLU(inplace=True)
            ]
            in_channels = out_channels
            out_channels = in_channels * 2
        
        # Residual blocks
        for _ in range(n_residual_blocks):
            model += [ResidualBlock(in_channels)]
        
        # Upsampling
        out_channels = in_channels // 2
        for _ in range(2):
            model += [
                nn.ConvTranspose2d(in_channels, out_channels, 3, stride=2, padding=1, output_padding=1),
                nn.InstanceNorm2d(out_channels),
                nn.ReLU(inplace=True)
            ]
            in_channels = out_channels
            out_channels = in_channels // 2
        
        # Output layer
        model += [nn.Conv2d(64, output_channels, 7, padding=3), nn.Tanh()]
        
        self.model = nn.Sequential(*model)

    def forward(self, x):
        return self.model(x)

class Discriminator(nn.Module):
    """PatchGAN discriminator"""
    def __init__(self, input_channels):
        super(Discriminator, self).__init__()
        
        # Initial convolution block
        model = [
            nn.Conv2d(input_channels, 64, 4, stride=2, padding=1),
            nn.LeakyReLU(0.2, inplace=True)
        ]
        
        # Downsampling layers
        in_channels = 64
        for i in range(3):
            out_channels = in_channels * 2
            model += [
                nn.Conv2d(in_channels, out_channels, 4, stride=2, padding=1),
                nn.InstanceNorm2d(out_channels),
                nn.LeakyReLU(0.2, inplace=True)
            ]
            in_channels = out_channels
        
        # Output layer
        model += [nn.Conv2d(out_channels, 1, 4, padding=1)]  # PatchGAN
        
        self.model = nn.Sequential(*model)

    def forward(self, x):
        return self.model(x)

class CycleGAN:
    def __init__(self, img_channels=3, gen_features=32, disc_features=64, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Generators: X -> Y and Y -> X
        self.G = Generator(img_channels, img_channels).to(self.device)
        self.F = Generator(img_channels, img_channels).to(self.device)
        
        # Discriminators: X and Y
        self.D_X = Discriminator(img_channels).to(self.device)
        self.D_Y = Discriminator(img_channels).to(self.device)
        
        # Optimizers
        self.optimizer_G = torch.optim.Adam(
            list(self.G.parameters()) + list(self.F.parameters()), lr=lr, betas=(0.5, 0.999)
        )
        self.optimizer_D = torch.optim.Adam(
            list(self.D_X.parameters()) + list(self.D_Y.parameters()), lr=lr, betas=(0.5, 0.999)
        )
        
        # Loss functions
        self.criterion_GAN = nn.MSELoss()  # LSGAN loss
        self.criterion_cycle = nn.L1Loss()
        self.criterion_identity = nn.L1Loss()
        
        self.lambda_cycle = 10.0
        self.lambda_identity = 0.5
    
    def train_step(self, real_X, real_Y):
        batch_size = real_X.size(0)
        
        # Adversarial ground truths
        real_labels = torch.ones(batch_size, 1, 30, 30, device=self.device)  # 30x30 comes from PatchGAN
        fake_labels = torch.zeros(batch_size, 1, 30, 30, device=self.device)
        
        # Train Generators
        self.optimizer_G.zero_grad()
        
        # Identity loss (optional: if you want G(y) to be y and F(x) to be x)
        same_X = self.F(real_X)
        loss_identity_X = self.criterion_identity(same_X, real_X) * self.lambda_identity
        
        same_Y = self.G(real_Y)
        loss_identity_Y = self.criterion_identity(same_Y, real_Y) * self.lambda_identity
        
        # GAN loss for G: X -> Y
        fake_Y = self.G(real_X)
        pred_fake_Y = self.D_Y(fake_Y)
        loss_GAN_G = self.criterion_GAN(pred_fake_Y, real_labels)
        
        # GAN loss for F: Y -> X
        fake_X = self.F(real_Y)
        pred_fake_X = self.D_X(fake_X)
        loss_GAN_F = self.criterion_GAN(pred_fake_X, real_labels)
        
        # Cycle loss X -> Y -> X
        recovered_X = self.F(fake_Y)
        loss_cycle_X = self.criterion_cycle(recovered_X, real_X) * self.lambda_cycle
        
        # Cycle loss Y -> X -> Y
        recovered_Y = self.G(fake_X)
        loss_cycle_Y = self.criterion_cycle(recovered_Y, real_Y) * self.lambda_cycle
        
        # Total generator loss
        loss_G = loss_GAN_G + loss_GAN_F + loss_cycle_X + loss_cycle_Y + loss_identity_X + loss_identity_Y
        loss_G.backward()
        self.optimizer_G.step()
        
        # Train Discriminators
        self.optimizer_D.zero_grad()
        
        # Real loss
        pred_real_X = self.D_X(real_X)
        loss_D_real_X = self.criterion_GAN(pred_real_X, real_labels)
        
        pred_real_Y = self.D_Y(real_Y)
        loss_D_real_Y = self.criterion_GAN(pred_real_Y, real_labels)
        
        # Fake loss
        fake_X_ = self.F(real_Y).detach()
        pred_fake_X_ = self.D_X(fake_X_)
        loss_D_fake_X = self.criterion_GAN(pred_fake_X_, fake_labels)
        
        fake_Y_ = self.G(real_X).detach()
        pred_fake_Y_ = self.D_Y(fake_Y_)
        loss_D_fake_Y = self.criterion_GAN(pred_fake_Y_, fake_labels)
        
        # Total discriminator loss
        loss_D_X = (loss_D_real_X + loss_D_fake_X) / 2
        loss_D_Y = (loss_D_real_Y + loss_D_fake_Y) / 2
        loss_D = loss_D_X + loss_D_Y
        
        loss_D.backward()
        self.optimizer_D.step()
        
        return {
            'loss_G': loss_G.item(),
            'loss_D': loss_D.item(),
            'loss_G_identity': (loss_identity_X + loss_identity_Y).item(),
            'loss_G_GAN': (loss_GAN_G + loss_GAN_F).item(),
            'loss_G_cycle': (loss_cycle_X + loss_cycle_Y).item(),
            'loss_D_real': (loss_D_real_X + loss_D_real_Y).item(),
            'loss_D_fake': (loss_D_fake_X + loss_D_fake_Y).item()
        }
    
    def generate(self, real_X, real_Y):
        """Generate fake samples for visualization"""
        self.G.eval()
        self.F.eval()
        
        with torch.no_grad():
            fake_Y = self.G(real_X)
            fake_X = self.F(real_Y)
            cycle_X = self.F(fake_Y)
            cycle_Y = self.G(fake_X)
        
        self.G.train()
        self.F.train()
        
        return fake_Y, fake_X, cycle_X, cycle_Y

# Example usage
cyclegan = CycleGAN(img_channels=3, gen_features=32, disc_features=64)

# Sample training data
sample_real_X = torch.randn(4, 3, 128, 128)
sample_real_Y = torch.randn(4, 3, 128, 128)

# Training step
losses = cyclegan.train_step(sample_real_X, sample_real_Y)
print(f"CycleGAN losses:")
for key, value in losses.items():
    print(f"  {key}: {value:.4f}")

# Generate samples
fake_Y, fake_X, cycle_X, cycle_Y = cyclegan.generate(sample_real_X, sample_real_Y)
print(f"Generated samples shapes:")
print(f"  fake_Y: {fake_Y.shape}")
print(f"  fake_X: {fake_X.shape}")
print(f"  cycle_X: {cycle_X.shape}")
print(f"  cycle_Y: {cycle_Y.shape}")
```

## 15. How do you implement a DiscoGAN for disentangled representation learning?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DiscoGANGenerator(nn.Module):
    """Generator for DiscoGAN with disentangled representations"""
    def __init__(self, input_channels, shared_dim=256, style_dim=64, n_residual_blocks=6):
        super(DiscoGANGenerator, self).__init__()
        
        # Shared encoder (content encoder)
        self.shared_encoder = nn.Sequential(
            nn.Conv2d(input_channels, 64, 7, padding=3),
            nn.InstanceNorm2d(64),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(64, 128, 3, stride=2, padding=1),
            nn.InstanceNorm2d(128),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(128, 256, 3, stride=2, padding=1),
            nn.InstanceNorm2d(256),
            nn.ReLU(inplace=True)
        )
        
        # Content decoder
        self.content_decoder = nn.Sequential(
            *[ResidualBlock(256) for _ in range(n_residual_blocks)],
            
            nn.ConvTranspose2d(256, 128, 3, stride=2, padding=1, output_padding=1),
            nn.InstanceNorm2d(128),
            nn.ReLU(inplace=True),
            
            nn.ConvTranspose2d(128, 64, 3, stride=2, padding=1, output_padding=1),
            nn.InstanceNorm2d(64),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(64, input_channels, 7, padding=3),
            nn.Tanh()
        )
        
        # Style encoder
        self.style_encoder = nn.Sequential(
            nn.Conv2d(input_channels, 64, 7, padding=3),
            nn.InstanceNorm2d(64),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(64, 128, 4, stride=2, padding=1),
            nn.InstanceNorm2d(128),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(128, 256, 4, stride=2, padding=1),
            nn.InstanceNorm2d(256),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(256, style_dim, 4, stride=4, padding=0),  # Global average pooling style
        )
        
        # Style-to-content mapping
        self.style_mapping = nn.Sequential(
            nn.Linear(style_dim, 256),
            nn.ReLU(inplace=True),
            nn.Linear(256, 256 * 4 * 4)  # For 4x4 spatial resolution
        )
    
    def encode_content(self, x):
        """Encode content representation"""
        return self.shared_encoder(x)
    
    def encode_style(self, x):
        """Encode style representation"""
        style = self.style_encoder(x)
        return style.view(style.size(0), -1)  # Flatten
    
    def decode(self, content, style=None):
        """Decode content with optional style injection"""
        if style is not None:
            # Map style to content space
            style_mapped = self.style_mapping(style)
            style_mapped = style_mapped.view(style.size(0), 256, 4, 4)
            # Upsample style to match content dimensions
            style_mapped = F.interpolate(style_mapped, size=content.size()[2:], mode='bilinear', align_corners=False)
            # Add style to content
            content = content + style_mapped
        
        return self.content_decoder(content)
    
    def forward(self, x, style=None):
        content = self.encode_content(x)
        if style is None:
            style = self.encode_style(x)
        return self.decode(content, style)

class DiscoGANDiscriminator(nn.Module):
    """Discriminator for DiscoGAN that can handle both domains"""
    def __init__(self, input_channels, shared_dim=256):
        super(DiscoGANDiscriminator, self).__init__()
        
        self.model = nn.Sequential(
            nn.Conv2d(input_channels, 64, 4, stride=2, padding=1),
            nn.LeakyReLU(0.2, inplace=True),
            
            nn.Conv2d(64, 128, 4, stride=2, padding=1),
            nn.InstanceNorm2d(128),
            nn.LeakyReLU(0.2, inplace=True),
            
            nn.Conv2d(128, 256, 4, stride=2, padding=1),
            nn.InstanceNorm2d(256),
            nn.LeakyReLU(0.2, inplace=True),
            
            nn.Conv2d(256, 1, 4, padding=1)  # PatchGAN
        )
    
    def forward(self, x):
        return self.model(x)

class DiscoGAN:
    def __init__(self, img_channels=3, shared_dim=256, style_dim=64, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Generators for both domains
        self.G_XY = DiscoGANGenerator(img_channels, shared_dim, style_dim).to(self.device)
        self.G_YX = DiscoGANGenerator(img_channels, shared_dim, style_dim).to(self.device)
        
        # Discriminators for both domains
        self.D_X = DiscoGANDiscriminator(img_channels).to(self.device)
        self.D_Y = DiscoGANDiscriminator(img_channels).to(self.device)
        
        # Optimizers
        self.optimizer_G = torch.optim.Adam(
            list(self.G_XY.parameters()) + list(self.G_YX.parameters()), lr=lr, betas=(0.5, 0.999)
        )
        self.optimizer_D = torch.optim.Adam(
            list(self.D_X.parameters()) + list(self.D_Y.parameters()), lr=lr, betas=(0.5, 0.999)
        )
        
        # Loss functions
        self.criterion_GAN = nn.MSELoss()
        self.criterion_content = nn.L1Loss()
        self.criterion_style = nn.L1Loss()
        self.criterion_cycle = nn.L1Loss()
        
        self.lambda_content = 1.0
        self.lambda_style = 1.0
        self.lambda_cycle = 10.0
    
    def train_step(self, real_X, real_Y):
        batch_size = real_X.size(0)
        
        # Adversarial targets
        real_labels = torch.ones(batch_size, 1, 16, 16, device=self.device)  # Patch size after convolutions
        fake_labels = torch.zeros(batch_size, 1, 16, 16, device=self.device)
        
        # Train Generators
        self.optimizer_G.zero_grad()
        
        # Encode content and style
        content_X = self.G_XY.encode_content(real_X)
        style_X = self.G_XY.encode_style(real_X)
        
        content_Y = self.G_YX.encode_content(real_Y)
        style_Y = self.G_YX.encode_style(real_Y)
        
        # Generate fake images
        fake_Y = self.G_XY.decode(content_X, style_Y)
        fake_X = self.G_YX.decode(content_Y, style_X)
        
        # Reconstruct
        rec_X = self.G_YX.decode(content_Y, style_X)  # content_Y + style_X -> fake_X -> content_Y + style_X
        rec_Y = self.G_XY.decode(content_X, style_Y)  # content_X + style_Y -> fake_Y -> content_X + style_Y
        
        # GAN losses
        pred_fake_Y = self.D_Y(fake_Y)
        loss_GAN_G_XY = self.criterion_GAN(pred_fake_Y, real_labels)
        
        pred_fake_X = self.D_X(fake_X)
        loss_GAN_G_YX = self.criterion_GAN(pred_fake_X, real_labels)
        
        # Content preservation loss
        content_fake_Y = self.G_XY.encode_content(fake_Y)
        content_fake_X = self.G_YX.encode_content(fake_X)
        loss_content = (self.criterion_content(content_fake_Y, content_X.detach()) +
                        self.criterion_content(content_fake_X, content_Y.detach())) * self.lambda_content
        
        # Style preservation loss
        style_fake_Y = self.G_XY.encode_style(fake_Y)
        style_fake_X = self.G_YX.encode_style(fake_X)
        loss_style = (self.criterion_style(style_fake_Y, style_Y.detach()) +
                      self.criterion_style(style_fake_X, style_X.detach())) * self.lambda_style
        
        # Cycle consistency loss
        cycle_X = self.G_YX.decode(content_Y, style_X)
        cycle_Y = self.G_XY.decode(content_X, style_Y)
        loss_cycle = (self.criterion_cycle(cycle_X, real_X) +
                      self.criterion_cycle(cycle_Y, real_Y)) * self.lambda_cycle
        
        # Total generator loss
        loss_G = (loss_GAN_G_XY + loss_GAN_G_YX + loss_content + loss_style + loss_cycle)
        loss_G.backward()
        self.optimizer_G.step()
        
        # Train Discriminators
        self.optimizer_D.zero_grad()
        
        # Real loss
        pred_real_X = self.D_X(real_X)
        pred_real_Y = self.D_Y(real_Y)
        loss_D_real = (self.criterion_GAN(pred_real_X, real_labels) +
                       self.criterion_GAN(pred_real_Y, real_labels)) / 2
        
        # Fake loss
        fake_X_ = self.G_YX(real_Y).detach()
        fake_Y_ = self.G_XY(real_X).detach()
        
        pred_fake_X_ = self.D_X(fake_X_)
        pred_fake_Y_ = self.D_Y(fake_Y_)
        loss_D_fake = (self.criterion_GAN(pred_fake_X_, fake_labels) +
                       self.criterion_GAN(pred_fake_Y_, fake_labels)) / 2
        
        # Total discriminator loss
        loss_D = (loss_D_real + loss_D_fake) / 2
        loss_D.backward()
        self.optimizer_D.step()
        
        return {
            'loss_G': loss_G.item(),
            'loss_D': loss_D.item(),
            'loss_G_GAN': (loss_GAN_G_XY + loss_GAN_G_YX).item(),
            'loss_G_content': loss_content.item(),
            'loss_G_style': loss_style.item(),
            'loss_G_cycle': loss_cycle.item(),
            'loss_D_real': loss_D_real.item(),
            'loss_D_fake': loss_D_fake.item()
        }
    
    def extract_representations(self, x, y):
        """Extract content and style representations"""
        self.G_XY.eval()
        self.G_YX.eval()
        
        with torch.no_grad():
            content_x = self.G_XY.encode_content(x)
            style_x = self.G_XY.encode_style(x)
            content_y = self.G_YX.encode_content(y)
            style_y = self.G_YX.encode_style(y)
        
        self.G_XY.train()
        self.G_YX.train()
        
        return content_x, style_x, content_y, style_y
    
    def generate_with_style_transfer(self, content_img, style_img):
        """Generate image with content from content_img and style from style_img"""
        self.G_XY.eval()
        self.G_YX.eval()
        
        with torch.no_grad():
            content = self.G_XY.encode_content(content_img)
            style = self.G_XY.encode_style(style_img)
            generated = self.G_XY.decode(content, style)
        
        self.G_XY.train()
        self.G_YX.train()
        
        return generated

# Example usage
disco_gan = DiscoGAN(img_channels=3, shared_dim=256, style_dim=64)

# Sample training data
sample_real_X = torch.randn(4, 3, 128, 128)
sample_real_Y = torch.randn(4, 3, 128, 128)

# Training step
losses = disco_gan.train_step(sample_real_X, sample_real_Y)
print(f"DiscoGAN losses:")
for key, value in losses.items():
    print(f"  {key}: {value:.4f}")

# Extract representations
content_x, style_x, content_y, style_y = disco_gan.extract_representations(sample_real_X, sample_real_Y)
print(f"Content X shape: {content_x.shape}")
print(f"Style X shape: {style_x.shape}")
print(f"Content Y shape: {content_y.shape}")
print(f"Style Y shape: {style_y.shape}")

# Style transfer example
generated_image = disco_gan.generate_with_style_transfer(sample_real_X[:1], sample_real_Y[:1])
print(f"Generated image with style transfer shape: {generated_image.shape}")
```

## 16. How do you implement a StarGAN for multi-domain image-to-image translation?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class StarGANGenerator(nn.Module):
    """Generator for StarGAN with multiple domains"""
    def __init__(self, img_channels=3, num_domains=5, conv_dim=64, repeat_num=6):
        super(StarGANGenerator, self).__init__()
        
        layers = []
        layers.append(nn.Conv2d(img_channels, conv_dim, kernel_size=7, stride=1, padding=3, bias=False))
        layers.append(nn.InstanceNorm2d(conv_dim, affine=True, track_running_stats=True))
        layers.append(nn.ReLU(inplace=True))
        
        # Down-sampling layers
        curr_dim = conv_dim
        for i in range(2):
            layers.append(nn.Conv2d(curr_dim, curr_dim*2, kernel_size=4, stride=2, padding=1, bias=False))
            layers.append(nn.InstanceNorm2d(curr_dim*2, affine=True, track_running_stats=True))
            layers.append(nn.ReLU(inplace=True))
            curr_dim = curr_dim * 2
        
        # Bottleneck layers
        for i in range(repeat_num):
            layers.append(ResidualBlock(curr_dim))
        
        # Up-sampling layers
        for i in range(2):
            layers.append(nn.ConvTranspose2d(curr_dim, curr_dim//2, kernel_size=4, stride=2, padding=1, bias=False))
            layers.append(nn.InstanceNorm2d(curr_dim//2, affine=True, track_running_stats=True))
            layers.append(nn.ReLU(inplace=True))
            curr_dim = curr_dim // 2
        
        layers.append(nn.Conv2d(curr_dim, img_channels, kernel_size=7, stride=1, padding=3, bias=False))
        layers.append(nn.Tanh())
        self.main = nn.Sequential(*layers)
        
        # Domain classification layer for reconstruction loss
        self.conv1 = nn.Conv2d(curr_dim, num_domains, kernel_size=1, stride=1, padding=0, bias=False)
    
    def forward(self, x, c):
        # Replicate spatially and concatenate domain information
        c = c.view(c.size(0), c.size(1), 1, 1)
        c = c.repeat(1, 1, x.size(2), x.size(3))
        x = torch.cat([x, c], dim=1)
        return self.main(x)

class StarGANDiscriminator(nn.Module):
    """Discriminator for StarGAN with multiple domains"""
    def __init__(self, img_channels=3, num_domains=5, conv_dim=64, repeat_num=6):
        super(StarGANDiscriminator, self).__init__()
        
        layers = []
        layers.append(nn.Conv2d(img_channels, conv_dim, kernel_size=4, stride=2, padding=1))
        layers.append(nn.LeakyReLU(0.01))
        
        curr_dim = conv_dim
        for i in range(1, repeat_num):
            layers.append(nn.Conv2d(curr_dim, curr_dim*2, kernel_size=4, stride=2, padding=1))
            layers.append(nn.LeakyReLU(0.01))
            curr_dim = curr_dim * 2
        
        kernel_size = 4 if img_channels == 3 else 2
        self.main = nn.Sequential(*layers)
        self.conv1 = nn.Conv2d(curr_dim, 1, kernel_size=3, stride=1, padding=1, bias=False)
        self.conv2 = nn.Conv2d(curr_dim, num_domains, kernel_size=kernel_size, bias=False)
    
    def forward(self, x):
        h = self.main(x)
        out_src = self.conv1(h)
        out_cls = self.conv2(h)
        return out_src, out_cls.view(out_cls.size(0), out_cls.size(1))

class StarGAN:
    def __init__(self, img_channels=3, num_domains=5, conv_dim=64, repeat_num=6, lr=0.0001):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.num_domains = num_domains
        
        self.G = StarGANGenerator(img_channels, num_domains, conv_dim, repeat_num).to(self.device)
        self.D = StarGANDiscriminator(img_channels, num_domains, conv_dim, repeat_num).to(self.device)
        
        self.g_optimizer = torch.optim.Adam(self.G.parameters(), lr, [0.5, 0.999])
        self.d_optimizer = torch.optim.Adam(self.D.parameters(), lr, [0.5, 0.999])
        
        self.criterion_adv = nn.MSELoss()
        self.criterion_cls = nn.CrossEntropyLoss()
        self.criterion_rec = nn.L1Loss()
        
        self.lambda_cls = 1
        self.lambda_rec = 10
        self.lambda_gp = 10
    
    def to_category(self, labels):
        """Convert label indices to one-hot vectors"""
        batch_size = labels.size(0)
        out = torch.zeros(batch_size, self.num_domains).to(self.device)
        indices = torch.LongTensor(labels).view(-1, 1)
        out.scatter_(1, indices, 1.0)
        return out
    
    def compute_gradient_penalty(self, real_images, fake_images):
        """Compute gradient penalty for WGAN-GP"""
        batch_size = real_images.size(0)
        alpha = torch.rand(batch_size, 1, 1, 1).to(self.device)
        interpolated = (alpha * real_images + (1 - alpha) * fake_images).requires_grad_(True)
        
        out_src, _ = self.D(interpolated)
        d_interpolated = torch.ones(interpolated.size(0), 1).to(self.device)
        
        gradients = torch.autograd.grad(outputs=out_src,
                                       inputs=interpolated,
                                       grad_outputs=d_interpolated,
                                       create_graph=True,
                                       retain_graph=True,
                                       only_inputs=True)[0]
        
        gradients = gradients.view(gradients.size(0), -1)
        gradient_penalty = torch.mean((gradients.norm(2, dim=1) - 1) ** 2)
        return gradient_penalty
    
    def train_step(self, real_X, label_X, label_Y):
        batch_size = real_X.size(0)
        
        # Convert labels to one-hot
        c_X = self.to_category(label_X)
        c_Y = self.to_category(label_Y)
        c_trg = c_Y
        
        # Compute loss with real images
        out_src, out_cls = self.D(real_X)
        d_loss_real = self.criterion_adv(out_src, torch.ones_like(out_src))
        d_loss_cls = self.criterion_cls(out_cls, label_X)
        
        # Generate fake images
        fake_X = self.G(real_X, c_trg)
        
        # Compute loss with fake images
        out_src, _ = self.D(fake_X.detach())
        d_loss_fake = self.criterion_adv(out_src, torch.zeros_like(out_src))
        
        # Compute gradient penalty
        gradient_penalty = self.compute_gradient_penalty(real_X.data, fake_X.data)
        
        # Discriminator total loss
        d_loss = d_loss_real + d_loss_fake + self.lambda_cls * d_loss_cls + self.lambda_gp * gradient_penalty
        
        self.d_optimizer.zero_grad()
        d_loss.backward()
        self.d_optimizer.step()
        
        # Train Generator
        self.g_optimizer.zero_grad()
        
        # Original-to-target domain
        fake_X = self.G(real_X, c_trg)
        out_src, out_cls = self.D(fake_X)
        
        g_loss_fake = self.criterion_adv(out_src, torch.ones_like(out_src))
        g_loss_cls = self.criterion_cls(out_cls, label_Y)
        
        # Target-to-original domain (cycle consistency)
        rec_X = self.G(fake_X, c_X)
        g_loss_rec = self.criterion_rec(real_X, rec_X)
        
        # Total generator loss
        g_loss = g_loss_fake + self.lambda_cls * g_loss_cls + self.lambda_rec * g_loss_rec
        
        g_loss.backward()
        self.g_optimizer.step()
        
        return d_loss.item(), g_loss.item()
    
    def generate(self, real_X, target_labels):
        """Generate images with specific target domains"""
        self.G.eval()
        
        with torch.no_grad():
            # Convert target labels to one-hot
            c_trg = self.to_category(target_labels)
            
            # Generate images
            fake_X = self.G(real_X, c_trg)
        
        self.G.train()
        return fake_X

# Example usage
num_domains = 5
stargan = StarGAN(img_channels=3, num_domains=num_domains, conv_dim=32, repeat_num=4)

# Sample training data
batch_size = 4
sample_real_X = torch.randn(batch_size, 3, 128, 128)
sample_labels_X = torch.randint(0, num_domains, (batch_size,))  # Domain labels for X
sample_labels_Y = torch.randint(0, num_domains, (batch_size,))  # Target domain labels

# Training step
d_loss, g_loss = stargan.train_step(sample_real_X, sample_labels_X, sample_labels_Y)
print(f"StarGAN Discriminator loss: {d_loss:.4f}")
print(f"StarGAN Generator loss: {g_loss:.4f}")

# Generate with specific target domain
target_domains = torch.tensor([0, 1, 2, 3])  # Target domains for each sample
generated = stargan.generate(sample_real_X, target_domains)
print(f"StarGAN generated images shape: {generated.shape}")
```

## 17. How do you implement a Progressive Growing GAN with training scheduling?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import DataLoader
import numpy as np

class ProgressiveGAN:
    def __init__(self, z_dim=512, n_features=16, img_channels=3, lr=0.001):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Initialize at lowest resolution (4x4)
        self.generator = Generator(z_dim, n_features, img_channels).to(self.device)
        self.discriminator = Discriminator(n_features, img_channels).to(self.device)
        
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0, 0.99))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0, 0.99))
        
        self.criterion = nn.BCELoss()
        self.z_dim = z_dim
    
    def train_phase(self, dataloader, phase, n_epochs, fade_length=10000):
        """
        Train for a specific resolution phase
        phase: 0=4x4, 1=8x8, 2=16x16, 3=32x32, 4=64x64, 5=128x128
        """
        print(f"Training at resolution: {4 * (2 ** phase)}x{4 * (2 ** phase)}")
        
        total_steps = 0
        for epoch in range(n_epochs):
            for i, real_data in enumerate(dataloader):
                real_data = real_data.to(self.device)
                batch_size = real_data.size(0)
                
                # Calculate alpha for smooth transition
                if phase > 0 and total_steps < fade_length:
                    alpha = total_steps / fade_length
                else:
                    alpha = 1.0
                
                # Train step
                d_loss, g_loss = self.train_step(real_data, alpha, phase)
                
                total_steps += 1
                
                if i % 100 == 0:
                    print(f"Epoch [{epoch+1}/{n_epochs}], Step [{i}], "
                          f"D: {d_loss:.4f}, G: {g_loss:.4f}, Alpha: {alpha:.3f}")
                
                # Stop if we've reached the fade length for this phase
                if phase > 0 and total_steps >= fade_length:
                    break
    
    def train_step(self, real_data, alpha, progress):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data, alpha, progress)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise, alpha, progress)
        fake_output = self.discriminator(fake_data.detach(), alpha, progress)
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data, alpha, progress)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()
    
    def progressive_training_schedule(self, dataset, epochs_per_phase=10, fade_length=10000):
        """Complete progressive training schedule"""
        # Define training phases with resolution and corresponding dataset
        phases = [
            {'resolution': 4, 'epochs': epochs_per_phase, 'batch_size': 64},
            {'resolution': 8, 'epochs': epochs_per_phase, 'batch_size': 32},
            {'resolution': 16, 'epochs': epochs_per_phase, 'batch_size': 16},
            {'resolution': 32, 'epochs': epochs_per_phase, 'batch_size': 8},
            {'resolution': 64, 'epochs': epochs_per_phase, 'batch_size': 4},
        ]
        
        for phase_idx, phase_config in enumerate(phases):
            print(f"\nStarting Phase {phase_idx}: Training at {phase_config['resolution']}x{phase_config['resolution']}")
            
            # Create dataloader for current resolution (you'd need to resize your dataset)
            # This is a simplified example - in practice you'd prepare a dataset at each resolution
            dataloader = DataLoader(dataset, batch_size=phase_config['batch_size'], shuffle=True)
            
            self.train_phase(
                dataloader, 
                phase=phase_idx, 
                n_epochs=phase_config['epochs'], 
                fade_length=fade_length
            )
            
            # Save model checkpoint
            self.save_checkpoint(f"progan_phase_{phase_idx}.pth")
    
    def save_checkpoint(self, filepath):
        """Save model checkpoint"""
        torch.save({
            'generator_state_dict': self.generator.state_dict(),
            'discriminator_state_dict': self.discriminator.state_dict(),
            'optimizer_G_state_dict': self.optimizer_G.state_dict(),
            'optimizer_D_state_dict': self.optimizer_D.state_dict(),
        }, filepath)
        print(f"Model checkpoint saved to {filepath}")
    
    def load_checkpoint(self, filepath):
        """Load model checkpoint"""
        checkpoint = torch.load(filepath, map_location=self.device)
        
        self.generator.load_state_dict(checkpoint['generator_state_dict'])
        self.discriminator.load_state_dict(checkpoint['discriminator_state_dict'])
        self.optimizer_G.load_state_dict(checkpoint['optimizer_G_state_dict'])
        self.optimizer_D.load_state_dict(checkpoint['optimizer_D_state_dict'])
        
        print(f"Model checkpoint loaded from {filepath}")
    
    def generate_samples(self, num_samples, resolution_phase, alpha=1.0):
        """Generate samples at specific resolution"""
        self.generator.eval()
        
        with torch.no_grad():
            noise = torch.randn(num_samples, self.z_dim, device=self.device)
            samples = self.generator(noise, alpha, resolution_phase)
        
        return samples

# Example usage with training schedule
def create_sample_dataset():
    """Create a sample dataset for demonstration"""
    from torch.utils.data import TensorDataset
    # Create random tensors as a placeholder dataset
    data = torch.randn(1000, 3, 4, 4)  # Small dataset for demo
    return TensorDataset(data)

progan = ProGAN(z_dim=512, n_features=16, img_channels=3, lr=0.001)

# Create sample dataset
sample_dataset = create_sample_dataset()

# Run progressive training (this would take a long time in practice)
# progan.progressive_training_schedule(sample_dataset, epochs_per_phase=5, fade_length=500)

# Generate samples at different phases
with torch.no_grad():
    samples_phase_0 = progan.generate_samples(4, resolution_phase=0, alpha=1.0)  # 4x4
    samples_phase_2 = progan.generate_samples(4, resolution_phase=2, alpha=1.0)  # 16x16
    samples_phase_4 = progan.generate_samples(4, resolution_phase=4, alpha=1.0)  # 64x64

print(f"Generated samples at 4x4: {samples_phase_0.shape}")
print(f"Generated samples at 16x16: {samples_phase_2.shape}")
print(f"Generated samples at 64x64: {samples_phase_4.shape}")
```

## 18. How do you implement a GAN with self-attention mechanism?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SelfAttention(nn.Module):
    """Self attention layer for SAGAN"""
    def __init__(self, in_dim):
        super(SelfAttention, self).__init__()
        self.chanel_in = in_dim
        
        self.query_conv = nn.Conv2d(in_channels=in_dim, out_channels=in_dim//8, kernel_size=1)
        self.key_conv = nn.Conv2d(in_channels=in_dim, out_channels=in_dim//8, kernel_size=1)
        self.value_conv = nn.Conv2d(in_channels=in_dim, out_channels=in_dim, kernel_size=1)
        self.gamma = nn.Parameter(torch.zeros(1))

        self.softmax = nn.Softmax(dim=-1)
    
    def forward(self, x):
        """
            inputs :
                x : input feature maps( B X C X W X H)
            returns :
                out : self attention value + input feature 
                attention: B X N X N (N is Width*Height)
        """
        m_batchsize, C, width, height = x.size()
        
        proj_query = self.query_conv(x).view(m_batchsize, -1, width*height).permute(0, 2, 1)  # B X CX(N)
        proj_key = self.key_conv(x).view(m_batchsize, -1, width*height)  # B X C x (*W*H)
        energy = torch.bmm(proj_query, proj_key)  # transpose check
        attention = self.softmax(energy)  # BX (N) X (N) 
        proj_value = self.value_conv(x).view(m_batchsize, -1, width*height)  # B X C X N

        out = torch.bmm(proj_value, attention.permute(0, 2, 1))
        out = out.view(m_batchsize, C, width, height)
        
        out = self.gamma*out + x
        return out, attention

class SAGANGenerator(nn.Module):
    """Generator with Self-Attention for SAGAN"""
    def __init__(self, z_dim=128, img_channels=3, attention_at_resolution=64):
        super(SAGANGenerator, self).__init__()
        
        self.init_size = 4
        self.attention_at_resolution = attention_at_resolution
        
        # Initial projection
        self.l1 = nn.Sequential(nn.Linear(z_dim, 1024 * self.init_size ** 2))
        
        # Upsampling blocks
        self.conv_blocks = nn.Sequential(
            # 4x4 -> 8x8
            nn.BatchNorm2d(1024),
            nn.Upsample(scale_factor=2),
            nn.Conv2d(1024, 512, 3, stride=1, padding=1),
            nn.BatchNorm2d(512),
            nn.ReLU(True),
            
            # 8x8 -> 16x16
            nn.Upsample(scale_factor=2),
            nn.Conv2d(512, 256, 3, stride=1, padding=1),
            nn.BatchNorm2d(256),
            nn.ReLU(True),
            
            # 16x16 -> 32x32
            nn.Upsample(scale_factor=2),
            nn.Conv2d(256, 128, 3, stride=1, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(True),
            
            # 32x32 -> 64x64 (with attention)
            nn.Upsample(scale_factor=2),
            nn.Conv2d(128, 64, 3, stride=1, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(True),
        )
        
        # Self-attention layer if needed at this resolution
        if attention_at_resolution == 64:
            self.attention = SelfAttention(64)
        else:
            self.attention = None
        
        # Final convolution
        self.conv_blocks_end = nn.Sequential(
            nn.Upsample(scale_factor=2),
            nn.Conv2d(64, 32, 3, stride=1, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(True),
            nn.Conv2d(32, img_channels, 3, stride=1, padding=1),
            nn.Tanh(),
        )

    def forward(self, z):
        out = self.l1(z)
        out = out.view(out.shape[0], 1024, self.init_size, self.init_size)
        out = self.conv_blocks(out)
        
        # Apply attention if available
        if self.attention:
            out, _ = self.attention(out)
        
        img = self.conv_blocks_end(out)
        return img

class SAGANDiscriminator(nn.Module):
    """Discriminator with Self-Attention for SAGAN"""
    def __init__(self, img_channels=3, attention_at_resolution=32):
        super(SAGANDiscriminator, self).__init__()
        
        self.attention_at_resolution = attention_at_resolution
        
        def discriminator_block(in_filters, out_filters, first_block=False):
            layers = []
            if not first_block:
                layers.append(nn.Conv2d(in_filters, out_filters, 4, 2, 1))
            else:
                layers.append(nn.Conv2d(in_filters, out_filters, 4, 2, 1))
            layers.append(nn.LeakyReLU(0.1, inplace=True))
            return layers

        self.conv_blocks = nn.Sequential(
            *discriminator_block(img_channels, 32, first_block=True),
            *discriminator_block(32, 64),
            *discriminator_block(64, 128),
            *discriminator_block(128, 256),
        )
        
        # Self-attention layer if needed at this resolution
        if attention_at_resolution == 32:  # After 32x32 layer
            self.attention = SelfAttention(256)
        else:
            self.attention = None
        
        # Final blocks after attention
        self.final_blocks = nn.Sequential(
            *discriminator_block(256, 512),
            nn.Conv2d(512, 1024, 4, 2, 1),
            nn.LeakyReLU(0.1, inplace=True),
        )
        
        self.adv_layer = nn.Linear(1024 * 4, 1)  # 4 is final spatial size after convolutions

    def forward(self, img):
        out = self.conv_blocks(img)
        
        # Apply attention if available
        if self.attention:
            out, _ = self.attention(out)
        
        out = self.final_blocks(out)
        
        out = out.view(out.shape[0], -1)
        validity = self.adv_layer(out)
        
        return validity

class SAGAN:
    def __init__(self, z_dim=128, img_channels=3, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = SAGANGenerator(z_dim, img_channels).to(self.device)
        self.discriminator = SAGANDiscriminator(img_channels).to(self.device)
        
        # Use Adam with different parameters as in original SAGAN paper
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0.0, 0.9))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0.0, 0.9))
        
        self.criterion = nn.BCEWithLogitsLoss()  # Use logits for stability
        self.z_dim = z_dim
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()
    
    def compute_hinge_loss(self, real_pred, fake_pred):
        """Compute hinge loss for more stable training"""
        d_loss_real = torch.mean(F.relu(1.0 - real_pred))
        d_loss_fake = torch.mean(F.relu(1.0 + fake_pred))
        d_loss = d_loss_real + d_loss_fake
        
        g_loss = -torch.mean(fake_pred)
        
        return d_loss, g_loss
    
    def train_step_with_hinge(self, real_data):
        """Train step with hinge loss for SAGAN"""
        batch_size = real_data.size(0)
        
        # Train Discriminator with hinge loss
        self.optimizer_D.zero_grad()
        
        real_output = self.discriminator(real_data)
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        
        d_loss, _ = self.compute_hinge_loss(real_output, fake_output)
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator with hinge loss
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        _, g_loss = self.compute_hinge_loss(real_output, fake_output)  # Only use g_loss
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# Example usage
sagan = SAGAN(z_dim=128, img_channels=3, lr=0.0002)

# Sample training data
sample_real_data = torch.randn(4, 3, 64, 64)

# Regular training step
d_loss, g_loss = sagan.train_step(sample_real_data)
print(f"SAGAN Discriminator loss: {d_loss:.4f}")
print(f"SAGAN Generator loss: {g_loss:.4f}")

# Training step with hinge loss
d_loss_hinge, g_loss_hinge = sagan.train_step_with_hinge(sample_real_data)
print(f"SAGAN with Hinge Discriminator loss: {d_loss_hinge:.4f}")
print(f"SAGAN with Hinge Generator loss: {g_loss_hinge:.4f}")

# Generate samples
with torch.no_grad():
    noise = torch.randn(5, 128, device=sagan.device)
    generated = sagan.generator(noise)
    print(f"SAGAN generated samples shape: {generated.shape}")
```

## 19. How do you implement a GAN with spectral normalization?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def l2normalize(v, eps=1e-12):
    return v / (v.norm() + eps)

class SpectralNorm(nn.Module):
    def __init__(self, module, name='weight', power_iterations=1):
        super(SpectralNorm, self).__init__()
        self.module = module
        self.name = name
        self.power_iterations = power_iterations
        if not self._made_params():
            self._make_params()

    def _update_u_v(self):
        u = getattr(self.module, self.name + "_u")
        v = getattr(self.module, self.name + "_v")
        w = getattr(self.module, self.name + "_bar")

        height = w.data.shape[0]
        for _ in range(self.power_iterations):
            v.data = l2normalize(torch.mv(torch.t(w.view(height,-1).data), u.data))
            u.data = l2normalize(torch.mv(w.view(height,-1).data, v.data))

        # sigma inverse
        sigma = u.dot(w.view(height, -1).mv(v))
        setattr(self.module, self.name, w / sigma.expand_as(w))

    def _made_params(self):
        try:
            u = getattr(self.module, self.name + "_u")
            v = getattr(self.module, self.name + "_v")
            w = getattr(self.module, self.name + "_bar")
            return True
        except AttributeError:
            return False

    def _make_params(self):
        w = getattr(self.module, self.name)

        height = w.data.shape[0]
        width = w.view(height, -1).data.shape[1]

        u = nn.Parameter(w.data.new(height).normal_(0, 1), requires_grad=False)
        v = nn.Parameter(w.data.new(width).normal_(0, 1), requires_grad=False)
        u.data = l2normalize(u.data)
        v.data = l2normalize(v.data)
        w_bar = nn.Parameter(w.data)

        del self.module._parameters[self.name]

        self.module.register_parameter(self.name + "_u", u)
        self.module.register_parameter(self.name + "_v", v)
        self.module.register_parameter(self.name + "_bar", w_bar)

    def forward(self, *args):
        self._update_u_v()
        return self.module.forward(*args)

class SpectralNormGenerator(nn.Module):
    def __init__(self, z_dim=128, img_channels=3):
        super(SpectralNormGenerator, self).__init__()
        
        self.init_size = 4
        self.l1 = nn.Sequential(nn.Linear(z_dim, 256 * self.init_size ** 2))
        
        self.conv_blocks = nn.Sequential(
            nn.BatchNorm2d(256),
            nn.Upsample(scale_factor=2),
            SpectralNorm(nn.Conv2d(256, 256, 3, stride=1, padding=1)),
            nn.BatchNorm2d(256),
            nn.ReLU(True),
            
            nn.Upsample(scale_factor=2),
            SpectralNorm(nn.Conv2d(256, 128, 3, stride=1, padding=1)),
            nn.BatchNorm2d(128),
            nn.ReLU(True),
            
            nn.Upsample(scale_factor=2),
            SpectralNorm(nn.Conv2d(128, 64, 3, stride=1, padding=1)),
            nn.BatchNorm2d(64),
            nn.ReLU(True),
            
            nn.Upsample(scale_factor=2),
            SpectralNorm(nn.Conv2d(64, 32, 3, stride=1, padding=1)),
            nn.BatchNorm2d(32),
            nn.ReLU(True),
            
            SpectralNorm(nn.Conv2d(32, img_channels, 3, stride=1, padding=1)),
            nn.Tanh(),
        )

    def forward(self, z):
        out = self.l1(z)
        out = out.view(out.shape[0], 256, self.init_size, self.init_size)
        img = self.conv_blocks(out)
        return img

class SpectralNormDiscriminator(nn.Module):
    def __init__(self, img_channels=3):
        super(SpectralNormDiscriminator, self).__init__()

        def discriminator_block(in_filters, out_filters, first_block=False):
            layers = []
            if not first_block:
                layers.append(SpectralNorm(nn.Conv2d(in_filters, out_filters, 4, 2, 1, bias=False)))
            else:
                layers.append(SpectralNorm(nn.Conv2d(in_filters, out_filters, 4, 2, 1, bias=False)))
            layers.append(nn.LeakyReLU(0.1, inplace=True))
            return layers

        self.conv_blocks = nn.Sequential(
            *discriminator_block(img_channels, 32, first_block=True),
            *discriminator_block(32, 64),
            *discriminator_block(64, 128),
            *discriminator_block(128, 256),
        )

        # The height and width of downsampled image
        ds_size = 128 // 2 ** 4
        self.adv_layer = nn.Sequential(
            SpectralNorm(nn.Conv2d(256, 1, ds_size, bias=False)),
            nn.Flatten(),
        )

    def forward(self, img):
        out = self.conv_blocks(img)
        validity = self.adv_layer(out)
        return validity

class SpectralNormGAN:
    def __init__(self, z_dim=128, img_channels=3, lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.generator = SpectralNormGenerator(z_dim, img_channels).to(self.device)
        self.discriminator = SpectralNormDiscriminator(img_channels).to(self.device)
        
        # Use Adam with parameters from SNGAN paper
        self.optimizer_G = torch.optim.Adam(self.generator.parameters(), lr=lr, betas=(0.0, 0.9))
        self.optimizer_D = torch.optim.Adam(self.discriminator.parameters(), lr=lr, betas=(0.0, 0.9))
        
        self.criterion = nn.BCEWithLogitsLoss()
        self.z_dim = z_dim
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=self.device)
        fake_labels = torch.zeros(batch_size, 1, device=self.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, self.z_dim, device=self.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        g_loss = self.criterion(fake_output, real_labels)  # Try to fool discriminator
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()
    
    def compute_hinge_loss(self, real_pred, fake_pred):
        """Compute hinge loss for spectral normalization"""
        d_loss_real = torch.mean(F.relu(1.0 - real_pred))
        d_loss_fake = torch.mean(F.relu(1.0 + fake_pred))
        d_loss = d_loss_real + d_loss_fake
        
        g_loss = -torch.mean(fake_pred)
        
        return d_loss, g_loss

# Example usage
sngan = SpectralNormGAN(z_dim=128, img_channels=3, lr=0.0002)

# Sample training data
sample_real_data = torch.randn(4, 3, 64, 64)

# Training step
d_loss, g_loss = sngan.train_step(sample_real_data)
print(f"SNGAN Discriminator loss: {d_loss:.4f}")
print(f"SNGAN Generator loss: {g_loss:.4f}")

# Generate samples
with torch.no_grad():
    noise = torch.randn(5, 128, device=sngan.device)
    generated = sngan.generator(noise)
    print(f"SNGAN generated samples shape: {generated.shape}")
```

## 20. How do you implement a GAN with different loss functions?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GANLossCalculator:
    """Different GAN loss functions"""
    
    @staticmethod
    def vanilla_loss(real_pred, fake_pred, real_labels, fake_labels):
        """Standard GAN loss (minimax)"""
        d_real_loss = F.binary_cross_entropy_with_logits(real_pred, real_labels)
        d_fake_loss = F.binary_cross_entropy_with_logits(fake_pred, fake_labels)
        d_loss = d_real_loss + d_fake_loss
        
        g_loss = F.binary_cross_entropy_with_logits(fake_pred, real_labels)  # Fool discriminator
        
        return d_loss, g_loss
    
    @staticmethod
    def least_squares_loss(real_pred, fake_pred, real_labels, fake_labels):
        """Least Squares GAN (LSGAN) loss"""
        # LSGAN uses squared loss instead of log loss
        d_real_loss = torch.mean((real_pred - real_labels) ** 2)
        d_fake_loss = torch.mean((fake_pred - fake_labels) ** 2)
        d_loss = 0.5 * (d_real_loss + d_fake_loss)
        
        g_loss = 0.5 * torch.mean((fake_pred - real_labels) ** 2)
        
        return d_loss, g_loss
    
    @staticmethod
    def hinge_loss(real_pred, fake_pred):
        """Hinge loss for GANs (used in SAGAN, SNGAN)"""
        d_real_loss = torch.mean(F.relu(1.0 - real_pred))
        d_fake_loss = torch.mean(F.relu(1.0 + fake_pred))
        d_loss = d_real_loss + d_fake_loss
        
        g_loss = -torch.mean(fake_pred)
        
        return d_loss, g_loss
    
    @staticmethod
    def relativistic_loss(real_pred, fake_pred, real_labels, fake_labels):
        """Relativistic GAN loss"""
        # Discriminator tries to make real data have higher scores than fake data
        d_loss = torch.mean((torch.sigmoid(real_pred - torch.mean(fake_pred)) - real_labels) ** 2) + \
                 torch.mean((torch.sigmoid(fake_pred - torch.mean(real_pred)) - fake_labels) ** 2)
        
        # Generator tries to make fake data have higher scores than real data
        g_loss = torch.mean((torch.sigmoid(real_pred - torch.mean(fake_pred)) - fake_labels) ** 2) + \
                 torch.mean((torch.sigmoid(fake_pred - torch.mean(real_pred)) - real_labels) ** 2)
        
        return d_loss, g_loss
    
    @staticmethod
    def wasserstein_loss(real_pred, fake_pred):
        """Wasserstein loss (for WGAN)"""
        d_loss = torch.mean(fake_pred) - torch.mean(real_pred)
        g_loss = -torch.mean(fake_pred)
        
        return d_loss, g_loss

class FlexibleGAN(nn.Module):
    def __init__(self, z_dim=128, img_channels=3, loss_type='vanilla'):
        super(FlexibleGAN, self).__init__()
        
        self.z_dim = z_dim
        self.loss_type = loss_type
        
        # Generator
        self.generator = nn.Sequential(
            nn.Linear(z_dim, 256),
            nn.ReLU(True),
            nn.Linear(256, 512),
            nn.ReLU(True),
            nn.Linear(512, 1024),
            nn.ReLU(True),
            nn.Linear(1024, img_channels * 32 * 32),
            nn.Tanh()
        )
        
        # Discriminator
        self.discriminator = nn.Sequential(
            nn.Linear(img_channels * 32 * 32, 1024),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Dropout(0.3),
            nn.Linear(1024, 512),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Dropout(0.3),
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Dropout(0.3),
            nn.Linear(256, 1)
        )
        
        # Loss calculator
        self.loss_calculator = GANLossCalculator()
    
    def forward(self, z):
        fake_images = self.generator(z)
        return fake_images
    
    def train_step(self, real_data, optimizer_G, optimizer_D, device='cpu'):
        batch_size = real_data.size(0)
        
        # Flatten real data
        real_data_flat = real_data.view(batch_size, -1)
        
        # Generate fake data
        noise = torch.randn(batch_size, self.z_dim, device=device)
        fake_data = self.generator(noise)
        fake_data_flat = fake_data.view(batch_size, -1)
        
        # Get predictions
        real_pred = self.discriminator(real_data_flat)
        fake_pred = self.discriminator(fake_data_flat.detach())
        
        # Calculate targets based on loss type
        if self.loss_type in ['vanilla', 'least_squares', 'relativistic']:
            real_labels = torch.ones(batch_size, 1, device=device)
            fake_labels = torch.zeros(batch_size, 1, device=device)
        else:  # wasserstein
            real_labels = None
            fake_labels = None
        
        # Calculate loss based on type
        if self.loss_type == 'vanilla':
            d_loss, g_loss = self.loss_calculator.vanilla_loss(
                real_pred, fake_pred, real_labels, fake_labels
            )
        elif self.loss_type == 'least_squares':
            d_loss, g_loss = self.loss_calculator.least_squares_loss(
                real_pred, fake_pred, real_labels, fake_labels
            )
        elif self.loss_type == 'hinge':
            d_loss, g_loss = self.loss_calculator.hinge_loss(real_pred, fake_pred)
        elif self.loss_type == 'relativistic':
            d_loss, g_loss = self.loss_calculator.relativistic_loss(
                real_pred, fake_pred, real_labels, fake_labels
            )
        elif self.loss_type == 'wasserstein':
            d_loss, g_loss = self.loss_calculator.wasserstein_loss(real_pred, fake_pred)
        else:
            raise ValueError(f"Unknown loss type: {self.loss_type}")
        
        # Train Discriminator
        optimizer_D.zero_grad()
        d_loss.backward()
        optimizer_D.step()
        
        # Train Generator
        optimizer_G.zero_grad()
        g_loss.backward()
        optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

class GANWithLossFunction:
    def __init__(self, z_dim=128, img_channels=3, loss_type='vanilla', lr=0.0002):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        self.model = FlexibleGAN(z_dim, img_channels, loss_type).to(self.device)
        self.loss_type = loss_type
        
        self.optimizer_G = torch.optim.Adam(self.model.generator.parameters(), lr=lr, betas=(0.5, 0.999))
        self.optimizer_D = torch.optim.Adam(self.model.discriminator.parameters(), lr=lr, betas=(0.5, 0.999))
        
        self.z_dim = z_dim
    
    def train_step(self, real_data):
        d_loss, g_loss = self.model.train_step(
            real_data, self.optimizer_G, self.optimizer_D, self.device
        )
        return d_loss, g_loss

# Example usage with different loss functions
loss_types = ['vanilla', 'least_squares', 'hinge']

for loss_type in loss_types:
    print(f"\nTraining with {loss_type.upper()} loss:")
    
    gan = GANWithLossFunction(z_dim=128, img_channels=3, loss_type=loss_type, lr=0.0002)
    
    # Sample training data
    sample_real_data = torch.randn(4, 3, 32, 32)
    
    # Training step
    d_loss, g_loss = gan.train_step(sample_real_data)
    print(f"  Discriminator loss: {d_loss:.4f}")
    print(f"  Generator loss: {g_loss:.4f}")
    
    # Generate samples
    with torch.no_grad():
        noise = torch.randn(3, 128, device=gan.device)
        generated = gan.model(noise)
        print(f"  Generated samples shape: {generated.shape}")

# Compare different losses
def compare_losses():
    print("\n" + "="*50)
    print("COMPARISON OF DIFFERENT GAN LOSS FUNCTIONS")
    print("="*50)
    
    print("1. Vanilla GAN Loss:")
    print("   - Uses sigmoid cross-entropy")
    print("   - Can suffer from vanishing gradients")
    print("   - Original GAN formulation")
    
    print("\n2. Least Squares GAN (LSGAN) Loss:")
    print("   - Uses squared error instead of log loss")
    print("   - Produces higher quality images")
    print("   - More stable training")
    
    print("\n3. Hinge Loss:")
    print("   - Used in SAGAN and SNGAN")
    print("   - Provides better gradient flow")
    print("   - More stable than cross-entropy")
    
    print("\n4. Relativistic GAN Loss:")
    print("   - Compares real vs fake as pairs")
    print("   - Can improve sample quality")
    print("   - More complex but potentially better")
    
    print("\n5. Wasserstein Loss (WGAN):")
    print("   - Provides meaningful loss metric")
    print("   - Better training stability")
    print("   - Requires weight clipping or gradient penalty")

compare_losses()