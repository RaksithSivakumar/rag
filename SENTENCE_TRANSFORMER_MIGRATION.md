# Sentence Transformer Migration for 15-Second Response Time

## Overview
This document outlines the migration from Google's embedding API to sentence transformers and the optimizations made to achieve 15-second response times.

## Key Changes Made

### 1. Dependencies Updated
- Added `sentence-transformers>=2.2.0` to `requirements.txt`
- Using `all-MiniLM-L6-v2` model (384 dimensions, very fast)
- Fallback to `paraphrase-MiniLM-L3-v2` if needed

### 2. Embedding System Migration (`utils/embed_utils.py`)
- **Replaced Google Embedding API** with sentence transformers
- **Model**: `all-MiniLM-L6-v2` (384 dimensions vs 1024)
- **Speed**: Local processing, no API calls
- **FAISS**: CPU-only implementation for maximum compatibility
- **Caching**: Enhanced caching for repeated embeddings
- **Batch Processing**: Optimized for 15-second target

### 3. Chunking Strategy Updated (`utils/pdf_utils.py`)
- **Page-based chunking**: Whole pages instead of small chunks
- **Chunk size**: Increased to 10,000 characters
- **Overlap**: Reduced to 0 for page-based processing
- **Fallback**: Large section splitting if page markers not found

### 4. Performance Optimizations (`main.py`)
- **Thread pools**: Optimized for sentence transformers
  - CPU executor: `min(cpu_count * 8, 64)`
  - IO executor: `min(cpu_count * 6, 48)`
  - Process executor: `min(cpu_count * 4, 32)`

- **Concurrency settings**:
  - Max concurrent questions: 4-16 (down from 32)
  - Batch sizes: 3-4 questions per batch
  - Timeouts: Reduced for faster processing

- **Timeouts**:
  - Document processing: 25 seconds (down from 35)
  - Search: 1.5 seconds (down from 2.0)
  - Answer generation: 6 seconds (down from 8.0)

### 5. Response Time Logging Removed
- Removed `processing_time` from API responses
- Removed `document_hash` from API responses
- Clean response format: `{"answers": [...]}`

## Performance Benefits

### Speed Improvements
1. **Local Embedding**: No API latency (was ~200-500ms per embedding)
2. **Smaller Dimensions**: 384 vs 1024 (faster vector operations)
3. **Page-based Chunks**: Fewer chunks to process
4. **Optimized Concurrency**: Better resource utilization

### Expected Response Times
- **Single question**: 2-5 seconds
- **Multiple questions**: 8-15 seconds
- **Large documents**: 10-15 seconds

## Technical Details

### Sentence Transformer Model
```python
model = SentenceTransformer('all-MiniLM-L6-v2')
# 384 dimensions, ~60MB model size
# Very fast inference (~100ms per embedding)
```

### FAISS Configuration
```python
DIMENSION = 384  # Reduced from 1024
index = faiss.IndexFlatIP(DIMENSION)  # CPU-only inner product for cosine similarity
```

### Chunking Strategy
```python
chunks = chunk_text(text, chunk_size=10000, overlap=0)
# Page-based chunks for better context
```

## Testing

Run the test script to verify the integration:
```bash
python test_sentence_transformers.py
```

## Migration Checklist

- [x] Update requirements.txt
- [x] Replace Google embedding API with sentence transformers
- [x] Update chunking strategy to page-based
- [x] Optimize thread pools and concurrency
- [x] Reduce timeouts for 15-second target
- [x] Remove response time logging
- [x] Update documentation
- [x] Create test script

## Notes

1. **First Run**: Sentence transformer model will be downloaded (~60MB)
2. **Memory Usage**: Model loaded in memory (~200MB)
3. **CPU Usage**: Local embedding generation uses CPU
4. **FAISS**: CPU-only implementation, no GPU dependencies
5. **Fallback**: Automatic fallback to smaller model if needed

## Troubleshooting

### Common Issues
1. **Model download fails**: Check internet connection
2. **Memory issues**: Reduce batch sizes in `embed_utils.py`
3. **Slow performance**: Check CPU usage and adjust thread pools

### Performance Tuning
- Adjust `max_workers` in thread pools
- Modify batch sizes in processing functions
- Tune similarity thresholds in search
- Adjust chunk sizes based on document types 