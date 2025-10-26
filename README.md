"""
FAST Telegram Chatbot with Neural Network (Optimized for Speed)
================================================================
Works on mobile CPU/GPU - Fast training with live progress!
"""

# ============= EASY CONFIGURATION =============
TELEGRAM_BOT_TOKEN = "8214971120:AAFB4GlgeyD_Q6oY6kRtg44GE9RUb5sexts"

PASTEBIN_URLS = [
    "https://huggingface.co/datasets/Rashikr/aimaks/raw/main/README.md",
    "https://huggingface.co/datasets/Rashikr/aimaks/raw/main/Aks",
]

# Fast training mode (reduced epochs for speed)
MODEL_SCALE = "tiny"  # Options: "tiny" (fastest), "small", "medium"

MODEL_CONFIGS = {
    "tiny": {  # Ultra-fast for testing (2-5 min training)
        "d_model": 128,
        "nhead": 4,
        "num_layers": 2,
        "dim_feedforward": 256,
        "vocab_size": 2000,
        "max_seq_length": 64,
        "batch_size": 32,
        "learning_rate": 0.001,
        "epochs": 3  # Very fast!
    },
    "small": {  # Fast training (5-10 min)
        "d_model": 192,
        "nhead": 4,
        "num_layers": 3,
        "dim_feedforward": 384,
        "vocab_size": 3000,
        "max_seq_length": 96,
        "batch_size": 24,
        "learning_rate": 0.0005,
        "epochs": 5
    },
    "medium": {  # Balanced (10-20 min)
        "d_model": 256,
        "nhead": 4,
        "num_layers": 4,
        "dim_feedforward": 512,
        "vocab_size": 4000,
        "max_seq_length": 128,
        "batch_size": 16,
        "learning_rate": 0.0003,
        "epochs": 8
    }
}
# ============= END CONFIGURATION =============

import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import requests
import re
import math
import pickle
import os
from collections import Counter
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
from datetime import datetime
import sys
import time

config = MODEL_CONFIGS[MODEL_SCALE]
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

print("="*60)
print("🚀 FAST Neural Telegram Chatbot")
print(f"📊 Model: {MODEL_SCALE.upper()}")
print(f"💻 Device: {device}")
print(f"⚡ Training Epochs: {config['epochs']}")
print("="*60)


class SimpleTokenizer:
    """Fast and simple tokenizer"""
    
    def __init__(self, vocab_size=2000):
        self.vocab_size = vocab_size
        self.word_to_idx = {'<PAD>': 0, '<SOS>': 1, '<EOS>': 2, '<UNK>': 3}
        self.idx_to_word = {0: '<PAD>', 1: '<SOS>', 2: '<EOS>', 3: '<UNK>'}
        self.word_freq = Counter()
        
    def build_vocab(self, text):
        """Build vocabulary quickly"""
        print("📖 Building vocabulary...")
        
        # Simple word tokenization
        words = re.findall(r'\w+|[^\w\s]', text.lower())
        self.word_freq.update(words)
        
        # Add most common words
        most_common = self.word_freq.most_common(self.vocab_size - 4)
        for word, _ in most_common:
            if word not in self.word_to_idx:
                idx = len(self.word_to_idx)
                self.word_to_idx[word] = idx
                self.idx_to_word[idx] = word
        
        print(f"✅ Vocabulary size: {len(self.word_to_idx)} words")
                
    def encode(self, text):
        """Encode text to indices"""
        words = re.findall(r'\w+|[^\w\s]', text.lower())
        return [self.word_to_idx.get(w, 3) for w in words]  # 3 = <UNK>
    
    def decode(self, indices):
        """Decode indices to text"""
        words = [self.idx_to_word.get(idx, '<UNK>') for idx in indices 
                 if idx not in [0, 1, 2]]  # Skip special tokens
        return ' '.join(words)


class FastTransformer(nn.Module):
    """Lightweight transformer for fast training"""
    
    def __init__(self, vocab_size, d_model, nhead, num_layers, dim_feedforward):
        super().__init__()
        self.d_model = d_model
        
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.pos_embedding = nn.Embedding(1000, d_model)
        
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model,
            nhead=nhead,
            dim_feedforward=dim_feedforward,
            dropout=0.1,
            batch_first=True,
            norm_first=True  # Faster training
        )
        self.transformer = nn.TransformerEncoder(encoder_layer, num_layers=num_layers)
        
        self.fc_out = nn.Linear(d_model, vocab_size)
        
        # Fast initialization
        nn.init.xavier_uniform_(self.embedding.weight)
        nn.init.xavier_uniform_(self.fc_out.weight)
        
    def forward(self, x):
        seq_len = x.size(1)
        positions = torch.arange(0, seq_len, device=x.device).unsqueeze(0)
        
        x = self.embedding(x) + self.pos_embedding(positions)
        x = self.transformer(x)
        return self.fc_out(x)
    
    def generate(self, prompt_ids, tokenizer, max_length=50, temperature=0.9):
        """Fast generation"""
        self.eval()
        with torch.no_grad():
            generated = prompt_ids.copy()
            
            for _ in range(max_length):
                # Limit context window for speed
                context = generated[-config['max_seq_length']:]
                x = torch.tensor([context], device=device)
                
                output = self(x)
                next_logits = output[0, -1, :] / temperature
                probs = torch.softmax(next_logits, dim=-1)
                
                # Sample token
                next_token = torch.multinomial(probs, 1).item()
                
                # Stop on EOS or padding
                if next_token in [0, 2]:
                    break
                    
                generated.append(next_token)
                
                if len(generated) > 100:  # Max response length
                    break
            
            return generated


class QuickDataset(Dataset):
    """Fast dataset creation"""
    
    def __init__(self, text, tokenizer, max_length):
        print("📝 Creating training pairs...")
        self.pairs = []
        self.tokenizer = tokenizer
        self.max_length = max_length
        
        # Split into sentences
        sentences = re.split(r'[.!?]+', text)
        sentences = [s.strip() for s in sentences if 10 < len(s.strip()) < 200]
        
        # Create Q-A pairs (consecutive sentences)
        for i in range(0, len(sentences) - 1, 2):  # Skip every other for speed
            if len(self.pairs) < 500:  # Limit pairs for fast training
                self.pairs.append((sentences[i], sentences[i + 1]))
        
        print(f"✅ Created {len(self.pairs)} training pairs")
        
    def __len__(self):
        return len(self.pairs)
    
    def __getitem__(self, idx):
        q, a = self.pairs[idx]
        
        # Encode
        q_ids = self.tokenizer.encode(q)[:self.max_length-1]
        a_ids = self.tokenizer.encode(a)[:self.max_length-2]
        
        # Add special tokens
        input_ids = q_ids + [1]  # Add SOS
        target_ids = [1] + a_ids + [2]  # SOS + answer + EOS
        
        # Pad
        input_ids += [0] * (self.max_length - len(input_ids))
        target_ids += [0] * (self.max_length - len(target_ids))
        
        return torch.tensor(input_ids[:self.max_length]), \
               torch.tensor(target_ids[:self.max_length])


def download_fast(urls):
    """Quick download with progress"""
    print(f"\n📥 Downloading {len(urls)} datasets...")
    all_text = []
    
    for i, url in enumerate(urls, 1):
        try:
            print(f"  [{i}/{len(urls)}] {url[:50]}...")
            resp = requests.get(url, timeout=20)
            resp.raise_for_status()
            text = resp.text
            all_text.append(text)
            print(f"  ✅ Downloaded {len(text):,} chars")
        except Exception as e:
            print(f"  ❌ Failed: {e}")
    
    combined = ' '.join(all_text)
    print(f"✅ Total: {len(combined):,} characters\n")
    return combined


def train_fast(model, dataloader, epochs, lr):
    """Fast training with live progress"""
    print("\n🎯 Starting Training...\n")
    
    model = model.to(device)
    optimizer = optim.AdamW(model.parameters(), lr=lr)
    criterion = nn.CrossEntropyLoss(ignore_index=0)
    
    start_time = time.time()
    
    for epoch in range(epochs):
        model.train()
        total_loss = 0
        batch_count = 0
        
        print(f"📊 Epoch {epoch+1}/{epochs}")
        
        for batch_idx, (src, tgt) in enumerate(dataloader):
            src, tgt = src.to(device), tgt.to(device)
            
            optimizer.zero_grad()
            
            # Forward
            output = model(src)
            loss = criterion(output.view(-1, output.size(-1)), tgt.view(-1))
            
            # Backward
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            optimizer.step()
            
            total_loss += loss.item()
            batch_count += 1
            
            # Progress every 5 batches
            if batch_idx % 5 == 0:
                progress = (batch_idx + 1) / len(dataloader) * 100
                print(f"  Progress: {progress:.0f}% | Loss: {loss.item():.4f}", end='\r')
        
        avg_loss = total_loss / batch_count
        elapsed = time.time() - start_time
        print(f"  ✅ Avg Loss: {avg_loss:.4f} | Time: {elapsed:.1f}s")
    
    total_time = time.time() - start_time
    print(f"\n🎉 Training Complete! Total time: {total_time:.1f}s\n")


def save_fast(model, tokenizer):
    """Quick save"""
    torch.save({
        'model': model.state_dict(),
        'config': config
    }, 'chatbot.pt')
    
    with open('tokenizer.pkl', 'wb') as f:
        pickle.dump(tokenizer, f)
    
    print("💾 Model saved!")


def load_fast(model):
    """Quick load"""
    if os.path.exists('chatbot.pt'):
        checkpoint = torch.load('chatbot.pt', map_location=device)
        model.load_state_dict(checkpoint['model'])
        print("✅ Loaded existing model")
        return True
    return False


# Global variables
model = None
tokenizer = None


async def start_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Start command"""
    await update.message.reply_text(
        "🤖 Hi! I'm a neural chatbot.\n"
        "💬 Send me any message!\n\n"
        "Commands:\n"
        "/start - This message\n"
        "/stats - Model info"
    )


async def stats_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Stats command"""
    params = sum(p.numel() for p in model.parameters())
    text = (
        f"📊 Model Stats:\n"
        f"• Scale: {MODEL_SCALE}\n"
        f"• Parameters: {params:,}\n"
        f"• Vocab: {len(tokenizer.word_to_idx)}\n"
        f"• Device: {device}"
    )
    await update.message.reply_text(text)


async def handle_msg(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle messages"""
    user_msg = update.message.text
    user_id = update.effective_user.id
    
    print(f"💬 User {user_id}: {user_msg}")
    
    try:
        # Encode and generate
        prompt_ids = tokenizer.encode(user_msg)
        response_ids = model.generate(prompt_ids, tokenizer, max_length=50)
        response = tokenizer.decode(response_ids)
        
        # Clean up
        response = response.strip()
        if not response or len(response) < 5:
            response = "I'm learning! Try asking something else."
        
        print(f"🤖 Bot: {response}")
        await update.message.reply_text(response)
        
        # Log
        with open('chat_log.txt', 'a', encoding='utf-8') as f:
            f.write(f"[{datetime.now()}] {user_id}: {user_msg}\n")
            f.write(f"[{datetime.now()}] Bot: {response}\n\n")
            
    except Exception as e:
        print(f"❌ Error: {e}")
        await update.message.reply_text("Sorry, something went wrong! 😅")


def main():
    """Main function"""
    global model, tokenizer
    
    try:
        # Download data
        text = download_fast(PASTEBIN_URLS)
        
        if len(text) < 100:
            print("❌ Not enough data! Check your URLs.")
            return
        
        # Build tokenizer
        tokenizer = SimpleTokenizer(vocab_size=config['vocab_size'])
        tokenizer.build_vocab(text)
        
        # Create dataset
        dataset = QuickDataset(text, tokenizer, config['max_seq_length'])
        
        if len(dataset) == 0:
            print("❌ No training pairs created!")
            return
        
        dataloader = DataLoader(
            dataset, 
            batch_size=config['batch_size'], 
            shuffle=True,
            num_workers=0  # Important for stability
        )
        
        # Initialize model
        print(f"🔧 Initializing {MODEL_SCALE} model...")
        model = FastTransformer(
            vocab_size=len(tokenizer.word_to_idx),
            d_model=config['d_model'],
            nhead=config['nhead'],
            num_layers=config['num_layers'],
            dim_feedforward=config['dim_feedforward']
        )
        
        params = sum(p.numel() for p in model.parameters())
        print(f"✅ Model ready! Parameters: {params:,}")
        
        # Load or train
        if not load_fast(model):
            train_fast(model, dataloader, config['epochs'], config['learning_rate'])
            save_fast(model, tokenizer)
        
        model = model.to(device)
        model.eval()
        
        # Test generation
        print("\n🧪 Testing model...")
        test_prompt = "hello"
        test_ids = tokenizer.encode(test_prompt)
        test_response_ids = model.generate(test_ids, tokenizer, max_length=20)
        test_response = tokenizer.decode(test_response_ids)
        print(f"Test input: '{test_prompt}'")
        print(f"Test output: '{test_response}'")
        
        # Start bot
        print("\n🚀 Starting Telegram bot...\n")
        app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
        
        app.add_handler(CommandHandler("start", start_cmd))
        app.add_handler(CommandHandler("stats", stats_cmd))
        app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_msg))
        
        print("✅ Bot is LIVE! Send messages on Telegram!\n")
        print("Press Ctrl+C to stop.\n")
        print("="*60)
        
        app.run_polling(allowed_updates=Update.ALL_TYPES)
        
    except KeyboardInterrupt:
        print("\n\n👋 Bot stopped by user")
    except Exception as e:
        print(f"\n❌ Fatal error: {e}")
        import traceback
        traceback.print_exc()


if __name__ == "__main__":
    main()
