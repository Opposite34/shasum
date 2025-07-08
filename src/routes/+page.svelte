<script lang="ts">
  import { createSHA256, createSHA512, createMD5 } from 'hash-wasm';
  import HashDisplay from '$lib/components/HashDisplay.svelte';
  import FileUpload from '$lib/components/FileUpload.svelte';

  let fileName = '';
  let md5 = '';
  let sha256 = '';
  let sha512 = '';
  let loadingMD5 = false;
  let loading256 = false;
  let loading512 = false;
  let progress = 0;
  let fileSize = 0;
  let eta = '';
  let statusLog = '';
  let startTime = 0;
  let processingTime = '';
  let hashInput = '';
  let hashCheckResult = '';

  // Optimized streaming hash implementation using hash-wasm for maximum performance
  async function hashFileInChunks(
    file: File
  ): Promise<{ md5: string; sha256: string; sha512: string }> {
    // Optimize chunk size for maximum throughput - larger chunks = better performance
    // Use 64MB chunks for maximum speed, as hash-wasm can handle large chunks efficiently
    const CHUNK_SIZE = 64 * 1024 * 1024; // 64MB chunks for maximum throughput
    const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
    statusLog = `Initializing hash computation (${totalChunks} chunks of ${CHUNK_SIZE / (1024 * 1024)} MB each)`;

    // Create hashers once and reuse them - this is a key optimization
    // Initialize all hashers at once to avoid overhead
    const [md5Hasher, sha256Hasher, sha512Hasher] = await Promise.all([
      createMD5(),
      createSHA256(),
      createSHA512()
    ]);

    // Pre-allocate FileReader to avoid creating new instances
    const fileReader = new FileReader();

    let processedBytes = 0;
    startTime = Date.now();

    // Process file using file.stream() for better performance when available
    if (file.stream && typeof file.stream === 'function') {
      try {
        const stream = file.stream();
        const reader = stream.getReader();
        let currentChunk = 0;

        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          currentChunk++;
          processedBytes += value.length;

          // Update all hashers with the chunk
          md5Hasher.update(value);
          sha256Hasher.update(value);
          sha512Hasher.update(value);

          progress = Math.round((processedBytes / file.size) * 99);

          // Calculate ETA and update status (less frequently for better performance)
          if (currentChunk % 5 === 0) {
            const elapsed = (Date.now() - startTime) / 1000; // seconds
            const rate = processedBytes / elapsed; // bytes per second
            const remaining = file.size - processedBytes;
            const etaSeconds = remaining / rate;

            if (etaSeconds > 0 && isFinite(etaSeconds)) {
              if (etaSeconds < 60) {
                eta = `${Math.round(etaSeconds)}s`;
              } else if (etaSeconds < 3600) {
                eta = `${Math.round(etaSeconds / 60)}m ${Math.round(etaSeconds % 60)}s`;
              } else {
                const hours = Math.floor(etaSeconds / 3600);
                const minutes = Math.floor((etaSeconds % 3600) / 60);
                eta = `${hours}h ${minutes}m`;
              }
            }

            // More detailed status with processing rate
            const rateMBps = (rate / (1024 * 1024)).toFixed(1);
            statusLog = `Processed ${(processedBytes / (1024 * 1024)).toFixed(1)}MB @ ${rateMBps} MB/s`;
          }

          // Yield control less frequently to maximize throughput
          if (currentChunk % 50 === 0) {
            await new Promise((resolve) => setTimeout(resolve, 0));
          }
        }
      } catch (streamError) {
        // Fallback to chunk-based processing if stream fails
        console.warn(
          'Stream processing failed, falling back to chunk-based processing:',
          streamError
        );
        return await hashFileInChunksLegacy(
          file,
          CHUNK_SIZE,
          md5Hasher,
          sha256Hasher,
          sha512Hasher
        );
      }
    } else {
      // Fallback for browsers that don't support file.stream()
      return await hashFileInChunksLegacy(file, CHUNK_SIZE, md5Hasher, sha256Hasher, sha512Hasher);
    }

    progress = 99;
    eta = 'Finalizing...';
    statusLog = 'Finalizing hash computation...';

    // Finalize the hashes (all at once for better performance)
    const [md5Hash, sha256Hash, sha512Hash] = await Promise.all([
      Promise.resolve(md5Hasher.digest()),
      Promise.resolve(sha256Hasher.digest()),
      Promise.resolve(sha512Hasher.digest())
    ]);

    progress = 100;
    eta = '';

    // Calculate total processing time
    const totalElapsed = (Date.now() - startTime) / 1000; // seconds
    const formatTime = (seconds: number) => {
      if (seconds < 60) {
        return `${seconds.toFixed(1)}s`;
      } else if (seconds < 3600) {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        return `${minutes}m ${remainingSeconds.toFixed(1)}s`;
      } else {
        const hours = Math.floor(seconds / 3600);
        const minutes = Math.floor((seconds % 3600) / 60);
        const remainingSeconds = seconds % 60;
        return `${hours}h ${minutes}m ${remainingSeconds.toFixed(1)}s`;
      }
    };

    processingTime = formatTime(totalElapsed);
    statusLog = `Hash computation completed successfully! Total time: ${processingTime}`;

    // Show completion message briefly
    setTimeout(() => {
      statusLog = '';
    }, 5000);

    return {
      md5: md5Hash,
      sha256: sha256Hash,
      sha512: sha512Hash
    };
  }

  // Legacy chunk-based processing for browsers without stream support
  async function hashFileInChunksLegacy(
    file: File,
    CHUNK_SIZE: number,
    md5Hasher: any,
    sha256Hasher: any,
    sha512Hasher: any
  ): Promise<{ md5: string; sha256: string; sha512: string }> {
    const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
    let processedBytes = 0;

    // Process file in chunks with optimized FileReader usage
    for (let offset = 0; offset < file.size; offset += CHUNK_SIZE) {
      const chunkSize = Math.min(CHUNK_SIZE, file.size - offset);
      const chunk = file.slice(offset, offset + chunkSize);
      const currentChunk = Math.ceil(offset / CHUNK_SIZE) + 1;

      // Use arrayBuffer() method if available (more efficient than FileReader)
      let data: Uint8Array;
      if (chunk.arrayBuffer && typeof chunk.arrayBuffer === 'function') {
        const buffer = await chunk.arrayBuffer();
        data = new Uint8Array(buffer);
      } else {
        // Fallback to FileReader for older browsers
        const buffer = await new Promise<ArrayBuffer>((resolve, reject) => {
          const reader = new FileReader();
          reader.onload = (e) => resolve(e.target?.result as ArrayBuffer);
          reader.onerror = () => reject(reader.error);
          reader.readAsArrayBuffer(chunk);
        });
        data = new Uint8Array(buffer);
      }

      // Update all hashers with the chunk
      md5Hasher.update(data);
      sha256Hasher.update(data);
      sha512Hasher.update(data);

      processedBytes += chunkSize;
      progress = Math.round((processedBytes / file.size) * 99);

      // Calculate ETA and update status
      const elapsed = (Date.now() - startTime) / 1000; // seconds
      const rate = processedBytes / elapsed; // bytes per second
      const remaining = file.size - processedBytes;
      const etaSeconds = remaining / rate;

      if (etaSeconds > 0 && isFinite(etaSeconds)) {
        if (etaSeconds < 60) {
          eta = `${Math.round(etaSeconds)}s`;
        } else if (etaSeconds < 3600) {
          eta = `${Math.round(etaSeconds / 60)}m ${Math.round(etaSeconds % 60)}s`;
        } else {
          const hours = Math.floor(etaSeconds / 3600);
          const minutes = Math.floor((etaSeconds % 3600) / 60);
          eta = `${hours}h ${minutes}m`;
        }
      }

      // More detailed status with processing rate
      const rateMBps = (rate / (1024 * 1024)).toFixed(1);
      statusLog = `Completed chunk ${currentChunk}/${totalChunks} ${(processedBytes / (1024 * 1024)).toFixed(1)}MB @ ${rateMBps} MB/s`;

      // Yield control to prevent blocking the UI every few chunks
      if (currentChunk % 5 === 0) {
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }

    // Finalize the hashes
    const md5Hash = md5Hasher.digest();
    const sha256Hash = sha256Hasher.digest();
    const sha512Hash = sha512Hasher.digest();

    return {
      md5: md5Hash,
      sha256: sha256Hash,
      sha512: sha512Hash
    };
  }

  // Handle file selection and compute checksums
  async function handleFileChange(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    if (!input.files?.length) return;

    const file = input.files[0];
    fileName = file.name;
    fileSize = file.size;
    md5 = '';
    sha256 = '';
    sha512 = '';
    hashInput = '';
    hashCheckResult = '';
    loadingMD5 = true;
    loading256 = true;
    loading512 = true;
    progress = 0;
    eta = '';
    processingTime = '';
    statusLog = `Selected file: ${file.name} (${(file.size / (1024 * 1024)).toFixed(1)} MB)`;

    try {
      // For files smaller than 128MB, use the shorthand methods which are faster
      // For larger files, use streaming to avoid memory issues
      const FILE_SIZE_THRESHOLD = 128 * 1024 * 1024; // 128MB

      if (file.size <= FILE_SIZE_THRESHOLD) {
        statusLog = 'Using optimized processing for small file...';
        startTime = Date.now();

        // Use file.arrayBuffer() if available (more efficient than FileReader)
        let buffer: ArrayBuffer;
        if (file.arrayBuffer && typeof file.arrayBuffer === 'function') {
          buffer = await file.arrayBuffer();
        } else {
          // Fallback to FileReader for older browsers
          buffer = await new Promise<ArrayBuffer>((resolve, reject) => {
            const reader = new FileReader();
            reader.onload = (e) => resolve(e.target?.result as ArrayBuffer);
            reader.onerror = () => reject(reader.error);
            reader.readAsArrayBuffer(file);
          });
        }

        const data = new Uint8Array(buffer);
        statusLog = 'Computing hashes using optimized algorithms...';

        // Import the shorthand functions for maximum performance
        const { md5: md5Func, sha256: sha256Func, sha512: sha512Func } = await import('hash-wasm');

        // Use Promise.all for parallel execution where possible
        const [md5Hash, sha256Hash, sha512Hash] = await Promise.all([
          md5Func(data),
          sha256Func(data),
          sha512Func(data)
        ]);

        md5 = md5Hash;
        sha256 = sha256Hash;
        sha512 = sha512Hash;

        const totalElapsed = (Date.now() - startTime) / 1000;
        processingTime =
          totalElapsed < 60
            ? `${totalElapsed.toFixed(1)}s`
            : `${Math.floor(totalElapsed / 60)}m ${(totalElapsed % 60).toFixed(1)}s`;
        statusLog = `Hash computation completed successfully! Total time: ${processingTime}`;

        progress = 100;

        // Show completion message briefly
        setTimeout(() => {
          statusLog = '';
        }, 5000);
      } else {
        // Use streaming hash-wasm for large files
        const result = await hashFileInChunks(file);
        md5 = result.md5;
        sha256 = result.sha256;
        sha512 = result.sha512;
      }
    } catch (e) {
      console.error('Error hashing file:', e);
      statusLog = `Error: ${e instanceof Error ? e.message : 'Unknown error occurred'}`;
    } finally {
      loadingMD5 = false;
      loading256 = false;
      loading512 = false;
      eta = '';
    }
  }

  // Format file size for display (moved to FileInfo component)
  // Copy hash to clipboard
  async function copyToClipboard(text: string) {
    try {
      await navigator.clipboard.writeText(text);
    } catch (err) {
      console.error('Failed to copy: ', err);
    }
  }
  $ : {
    if (hashInput.trim().toLowerCase().length === 64 && sha256) {
      hashCheckResult = hashInput.trim().toLowerCase() === sha256.toLowerCase() ? 'valid' : 'invalid'
    } else if (hashInput.trim().toLowerCase().length === 128 && sha512) {
      hashCheckResult = hashInput.trim().toLowerCase() === sha512.toLowerCase() ? 'valid' : 'invalid'
    } else if (hashInput.trim().toLowerCase().length === 32 && md5) {
      hashCheckResult = hashInput.trim().toLowerCase() === md5.toLowerCase() ? 'valid' : 'invalid'
    } else {
      hashCheckResult = ''
    }
  }
</script>

<main
  class="absolute inset-0 flex h-full min-h-screen w-full
flex-col items-center justify-center bg-[radial-gradient(#18181b_1px,transparent_1px)]
[background-size:16px_16px] p-4"
>
  <!-- No JavaScript warning -->
  <noscript>
    <div class="mb-6 rounded-lg border border-red-700 bg-red-900 p-4">
      <h2 class="mb-2 text-xl font-bold text-red-200">JavaScript Required</h2>
      <pre class="text-red-300">
Unsurprisingly, a complex task like this requires JavaScript to function properly.
If you cannot use JavaScript, please use your systems terminal to generate the hashes.
Linux: [sha256, sha512, md5]sum /path/to/your/file.iso
Windows: CertUtil -hashfile C:\path\to\your\file.iso [SHA256, SHA512, MD5]
MacOS: shasum -a [256, 512] /path/to/your/file.iso

If you disabled JavaScript, this is your fault.
      </pre>
    </div>
  </noscript>

  <div class="grid w-full max-w-4xl grid-cols-1 gap-6 md:grid-cols-2">
    <FileUpload
      {fileName}
      {fileSize}
      {progress}
      {eta}
      {statusLog}
      {processingTime}
      isProcessing={loadingMD5 || loading256 || loading512}
      onFileChange={handleFileChange}
    />

    <div class="rounded-lg bg-zinc-900 p-6 shadow-lg">
      <h2 class="mb-4 text-center text-xl font-bold">Checksums</h2>

      <div class="space-y-4">
        <HashDisplay label="SHA-256" hash={sha256} loading={loading256} onCopy={copyToClipboard} />

        <HashDisplay label="SHA-512" hash={sha512} loading={loading512} onCopy={copyToClipboard} />

        <HashDisplay label="MD5" hash={md5} loading={loadingMD5} onCopy={copyToClipboard} />
      </div>

      <div class="mt-2">
        <p class="text-md text-zinc-50">Hash Checker</p>
        <p class="mb-2 text-sm text-zinc-400">Auto-detects hash type based on length</p>
        <div class="relative mt-2">
          <input
            type="text"
            placeholder="Enter hash to check"
            class="w-full rounded border-2 bg-zinc-800 px-3 py-2 pr-10 text-sm
text-zinc-300 placeholder:text-zinc-500 focus:ring-0 focus:outline-none"
            class:border-green-500={hashCheckResult === "valid"}
            class:border-red-500={hashCheckResult === "invalid"}
            class:border-zinc-700={hashCheckResult === ""}
            bind:value={hashInput}
            autocomplete="off"
          />

          <!-- Status icon -->
          <div class="absolute top-1/2 right-3 flex -translate-y-1/2 items-center">
            {#if hashCheckResult === 'valid'}
              <span
                class="icon-[material-symbols--check-circle] text-xl text-green-500"
                title="Hash matches!"
              ></span>
            {:else if hashCheckResult === 'invalid'}
              <span
                class="icon-[material-symbols--cancel] text-xl text-red-500"
                title="Hash does not match"
              ></span>
            {/if}
          </div>
        </div>
      </div>
    </div>
  </div>

  <footer class="mt-4 text-center text-zinc-500">
    <p class="flex items-center justify-center gap-1 text-sm">
      <span class="icon-[material-symbols--info-outline] text-2xl"></span>
      All processing is done locally in your browser. This may not work on certain browsers or devices.
    </p>
    <p class="flex items-center justify-center gap-1 text-sm">
      <span class="icon-[material-symbols--warning] text-2xl"></span>
      This is generally slower than using your host system's commands for generating hashes.
    </p>
    <p class="gap-1 text-sm">
      Powered by <a
        href="https://www.npmjs.com/package/hash-wasm"
        class="text-blue-500 hover:underline">hash-wasm</a
      >.
    </p>
    <p class="m-2 text-xs text-zinc-400">
      Designed by accuratelinuxgraphs.com - Empowering websites with accurate CSS since 1995.
    </p>
  </footer>
</main>
