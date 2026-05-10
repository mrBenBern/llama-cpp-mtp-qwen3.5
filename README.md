

```markdown
# llama.cpp MTP для Qwen — RTX 3060 12GB

Готовый Docker-образ с поддержкой Multi-Token Prediction (MTP) для моделей Qwen 3.5 на видеокартах NVIDIA.

**Собрано из ветки `mtp-clean` ([PR #22673](https://github.com/ggml-org/llama.cpp/pull/22673)).**

[![Docker Pulls](https://img.shields.io/badge/docker-ghcr.io%2Fmrbenbern%2Fllama--cpp--mtp--qwen-blue)](https://github.com/mrbenbern/llama-cpp-mtp-qwen/pkgs/container/llama-cpp-mtp-qwen)

## Быстрый старт

```bash
docker pull ghcr.io/mrbenbern/llama-cpp-mtp-qwen:latest

docker run --gpus all -v ./models:/models -p 8080:8080 \
  ghcr.io/mrbenbern/llama-cpp-mtp-qwen:latest \
  -m /models/Qwen3.5-9B-SOMPOA-heresy-MTP-Q4_K_M.gguf \
  --spec-type mtp --spec-draft-n-max 3 --parallel 1 \
  -ngl 999 -fa on -c 98304
```

## Результаты на RTX 3060 12GB

- Модель: Qwen 3.5 9B SOMPOA Heresy MTP (Q4_K_M, ~5.4 ГБ)
- Контекст: 98k токенов
- Скорость: 65+ t/s
- MTP acceptance rate: 70%+
- VRAM занято: ~7.3 ГБ
Образ: `docker pull ghcr.io/mrbenbern/llama-cpp-mtp-qwen:latest`

[Docker-образ на GHCR](https://github.com/mrBenBern?tab=packages)

[Репозиторий на GitHub](https://github.com/mrBenBern/llama-cpp-mtp-qwen3.5)

## Сборка из исходников

Если хочешь собрать образ сам:

```dockerfile
FROM nvidia/cuda:12.6.0-devel-ubuntu22.04 AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential cmake git libcurl4-openssl-dev ca-certificates && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

ENV CUDA_DOCKER_ARCH=all
ENV LIBRARY_PATH="/usr/local/cuda/lib64/stubs:$LIBRARY_PATH"

RUN git clone --depth 1 --branch mtp-clean https://github.com/am17an/llama.cpp.git . && \
    cmake -B build \
    -DGGML_CUDA=ON \
    -DGGML_LTO=OFF \
    -DCMAKE_EXE_LINKER_FLAGS="-lcuda -L/usr/local/cuda/lib64/stubs" && \
    cmake --build build --config Release -j $(nproc)

FROM nvidia/cuda:12.6.0-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 ca-certificates && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/build/bin/llama-server /llama-server
COPY --from=builder /app/build/bin/*.so* /usr/lib/

RUN chmod +x /llama-server
ENV LD_LIBRARY_PATH=/usr/lib:/usr/local/cuda/lib64

EXPOSE 8080
ENTRYPOINT ["/llama-server"]
```

Сборка:
```bash
docker build -t llama-cpp-mtp-qwen .
```

## Проблемы и решения

При сборке столкнулись с ошибками:
- **LTO линковка (67%)** → `-DGGML_LTO=OFF`
- **Не найден nvcc** → добавлен `nvidia-cuda-toolkit`
- **undefined reference к CUDA** → `-DCMAKE_EXE_LINKER_FLAGS="-lcuda -L/usr/local/cuda/lib64/stubs"`
- **Конфликт PR с мастером** → клонирование `mtp-clean` вместо merge

## Модели

Протестировано с:
- [Qwen3.5-9B-SOMPOA-heresy-MTP](https://huggingface.co/MuXodious/Qwen3.5-9B-SOMPOA-heresy-MTP-GGUF)
- Должно работать с любой Qwen 3.5/3.6 MTP моделью
```

