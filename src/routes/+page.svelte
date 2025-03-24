<script lang="ts">
  // No need to import runes in Svelte 5, they are built-in
  import { page } from '$app/stores';
  
  interface ArenaBlock {
    id: number;
    title?: string;
    class: string;
    content?: string;
    source?: {
      url: string;
    };
    image?: {
      display: {
        url: string;
      }
    };
    attachment?: {
      url?: string;
      content_type?: string;
    };
  }
  
  // State using runes with $ prefix directly
  let channelSlug = $state(''); // Empty default channel
  let contents = $state<ArenaBlock[]>([]);
  let loading = $state(false); // Start with loading false since we're not fetching initially
  let error = $state<string | null>(null);
  let isMobile = $state(false); // Default value for server
  let columnCount = $state(5); // Default to 5 columns initially
  let minColumns = $state(1); // Initialize with a default value
  let maxColumns = $state(10); // Default max for desktop
  let totalPages = $state(1);
  let currentPage = $state(1);
  let isLoadingMore = $state(false);
  let hasMorePages = $state(true);
  let selectedImage = $state<ArenaBlock | null>(null);
  
  // Then add this to initialize browser-specific values
  $effect(() => {
    // This will only run in the browser
    if (typeof window !== 'undefined') {
      isMobile = window.innerWidth < 768;
      minColumns = isMobile ? 1 : 2; // Set min columns based on device
      columnCount = isMobile ? 2 : 5; // Set appropriate column count based on device
    }
  });

  // Fix the state reference warnings by using a function for the resize handler
  $effect(() => {
    if (typeof window === 'undefined') return;
    
    const handleResize = () => {
      const newIsMobile = window.innerWidth < 768;
      isMobile = newIsMobile;
      minColumns = newIsMobile ? 1 : 2; // Update min columns on resize
      
      if (newIsMobile) {
        if (columnCount > 5) {
          columnCount = 5; // Limit to 5 columns on mobile
        }
      } else {
        if (columnCount < 2) {
          columnCount = 2; // Ensure at least 2 columns on desktop
        }
      }
    };
    
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });

  // Derived value for grid template columns
  let gridColumnsStyle = $derived(`repeat(${columnCount}, 1fr)`);
  
  // Parse channel from URL on initial load and when URL changes
  $effect(() => {
    // Get the channel from URL parameters
    const urlChannel = $page.url.searchParams.get('channel');
    if (urlChannel) {
      channelSlug = urlChannel;
    }
  });
  
  // Function to update URL with current channel
  function updateUrl() {
    const url = new URL(window.location.href);
    
    // Update or remove channel parameter
    if (channelSlug) {
      url.searchParams.set('channel', channelSlug);
    } else {
      url.searchParams.delete('channel');
    }
    
    // Update URL without reloading the page
    window.history.pushState({}, '', url.toString());
  }
  
  // Function to fetch channel metadata
  async function fetchChannelMetadata() {
    const response = await fetch(`https://api.are.na/v2/channels/${channelSlug}`);
    
    if (!response.ok) {
        throw new Error(`Failed to fetch channel metadata: ${response.statusText}`);
    }
    
    const data = await response.json();
    console.log('Channel metadata:', {
        length: data.length,
        counts: data.counts
    });
    return data;
  }
  
  // Function to fetch a single page of channel data
  async function fetchChannelPage(page: number) {
    const PER_PAGE = 100; // We can use 100 per page to reduce number of requests
    const response = await fetch(
        `https://api.are.na/v2/channels/${channelSlug}/contents?` + 
        `page=${page}&per=${PER_PAGE}&direction=desc&sort=position`
    );
    
    if (!response.ok) {
        throw new Error(`Failed to fetch channel page ${page}: ${response.statusText}`);
    }
    
    return await response.json();
  }
  
  // Add this function to check if we're near bottom
  function isNearBottom(element: Element, threshold = 150) {
    const { scrollTop, scrollHeight, clientHeight } = element;
    return scrollHeight - (scrollTop + clientHeight) < threshold;
  }

  // Add scroll handler effect
  $effect(() => {
    const container = document.querySelector('.scroll-container');
    if (!container) return;

    const handleScroll = async () => {
        if (
            !isLoadingMore && 
            hasMorePages && 
            !loading && 
            isNearBottom(container)
        ) {
            await loadNextPage();
        }
    };

    container.addEventListener('scroll', handleScroll);
    return () => container.removeEventListener('scroll', handleScroll);
  });

  // Add function to load next page
  async function loadNextPage() {
    if (isLoadingMore || !hasMorePages) return;
    
    try {
        isLoadingMore = true;
        currentPage++;
        
        console.log(`Loading page ${currentPage} of ${totalPages}`);
        const pageData = await fetchChannelPage(currentPage);
        
        const imageBlocks = pageData.contents.filter((block: ArenaBlock) => 
            block.class === 'Image' || 
            (block.attachment && block.attachment.content_type?.startsWith('image/'))
        );
        
        contents = [...contents, ...imageBlocks];
        console.log(`Loaded ${imageBlocks.length} more images. Total: ${contents.length}`);
        
        hasMorePages = currentPage < totalPages;
    } catch (error) {
        console.error('Error loading more images:', error);
    } finally {
        isLoadingMore = false;
    }
  }

  // Modify the fetchChannelData function
  async function fetchChannelData() {
    loading = true;
    error = null;
    contents = [];
    currentPage = 0;
    hasMorePages = true;
    
    try {
        updateUrl();
        
        // Get channel metadata
        const metadata = await fetchChannelMetadata();
        const totalItems = metadata.length;
        const PER_PAGE = 100;
        totalPages = Math.ceil(totalItems / PER_PAGE);
        
        console.log(`Channel has ${totalItems} total items across ${totalPages} pages`);
        
        // Load just the first page
        await loadNextPage();
        
    } catch (err) {
        if (err instanceof Error) {
            error = err.message;
        } else {
            error = 'An unknown error occurred';
        }
        console.error('Error fetching channel data:', err);
        contents = [];
    } finally {
        loading = false;
    }
  }
  
  // Function to decrease column count
  function decreaseColumns() {
    if (columnCount > minColumns) {
      columnCount--;
      // Force UI update by explicitly setting gridColumnsStyle
      gridColumnsStyle = `repeat(${columnCount}, 1fr)`;
    }
  }
  
  // Function to increase column count
  function increaseColumns() {
    const mobileMax = 5;
    const desktopMax = maxColumns;
    const currentMax = isMobile ? mobileMax : desktopMax;
        
    if (columnCount < currentMax) {
      columnCount++;
      // Force UI update by explicitly setting gridColumnsStyle
      gridColumnsStyle = `repeat(${columnCount}, 1fr)`;
    }
  }
  
  // Function to randomize the order of images (keep this function for internal use)
  function randomizeImages() {
    // Create a copy of the contents array and shuffle it
    const shuffled = [...contents];
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    contents = shuffled;
  }
  
  // Handle input change with proper typing
  function handleColumnChange(e: Event) {
    const target = e.target as HTMLInputElement;
    if (target && target.value) {
      const newValue = parseInt(target.value);
      if (!isNaN(newValue) && newValue >= minColumns && newValue <= maxColumns) {
        columnCount = newValue;
        updateUrl();
      }
    }
  }

  // Handle channel slug change
  function handleChannelChange(e: Event) {
    const target = e.target as HTMLInputElement;
    channelSlug = target.value || '';
    if (!target.value) {
      // If channel is cleared, update URL to remove it
      updateUrl();
    }
  }
  
  // Remove touch event handlers for pinch zoom
  
  // Fetch data on component mount and when channelSlug changes
  $effect(() => {
    if (channelSlug) { // Only fetch if we have a channel slug
      fetchChannelData();
    }
  });
  
  // Remove meta viewport tag to disable zooming

  function openImage(block: ArenaBlock) {
    selectedImage = block;
  }

  function closeImage() {
    selectedImage = null;
  }

  // Add escape key handler
  $effect(() => {
    if (typeof window === 'undefined') return;

    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && selectedImage) {
        closeImage();
      }
    };

    window.addEventListener('keydown', handleEscape);
    return () => window.removeEventListener('keydown', handleEscape);
  });
</script>

<div class="flex flex-col h-screen bg-black text-white overflow-hidden">
  <!-- Header controls -->
  <div class="flex items-center justify-between p-2 bg-black text-xs font-mono">
    <div class="flex items-center">
      <span>are.na: &nbsp;</span>
      <input 
        type="text" 
        value={channelSlug}
        onchange={handleChannelChange}
        placeholder="channel slug"
        class="bg-transparent border-none text-xs font-mono focus:outline-none focus:ring-0 w-40 px-0 py-0 opacity-75 italic"
      />
    </div>
    <div class="flex items-center">
      <button 
        type="button"
        onclick={(e) => { e.preventDefault(); decreaseColumns(); }}
        class="text-xs font-mono hover:text-gray-400 px-1 pr-5"
      >
        -
      </button>
      <button 
        type="button"
        onclick={(e) => { e.preventDefault(); increaseColumns(); }}
        class="text-xs font-mono hover:text-gray-400 px-1 pl-5"
      >
        +
      </button>
    </div>
  </div>
  
  <!-- Main grid content -->
  <div class="flex-1 overflow-y-auto overflow-x-hidden p-0 relative scroll-container">
    {#if loading}
      <div class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-75">
        <div class="text-xl">Loading...</div>
      </div>
    {:else if error}
      <div class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-75">
        <div class="text-xl text-red-500">Error: {error}</div>
      </div>
    {:else if !channelSlug || contents.length === 0}
      <div class="absolute inset-0 flex items-center justify-center bg-black 50vw">
        <div class="text-center">
          <div class="text-sm opacity-75 font-mono">enter a channel slug above and click load to view</div>
        </div>
      </div>
    {:else}
      <div 
        class="grid gap-0 w-full" 
        style:grid-template-columns={gridColumnsStyle}
      >
        {#each contents as block}
          <div class="aspect-square w-full overflow-hidden">
            <button 
              class="w-full h-full p-0 m-0 border-0 bg-transparent"
              onclick={() => openImage(block)}
            >
              <img 
                src={(block.image && block.image.display.url) || 
                    (block.attachment && block.attachment.url) || 
                    'https://via.placeholder.com/300'} 
                alt={block.title || 'Arena Block'} 
                class="w-full h-full object-cover hover:opacity-90 transition-opacity"
                loading="lazy"
              />
            </button>
          </div>
        {/each}
      </div>
      
      {#if isLoadingMore}
        <div class="w-full p-4 text-center text-sm opacity-75 font-mono">
          Loading more images...
        </div>
      {/if}
    {/if}

    <!-- Add modal markup -->
    {#if selectedImage}
      <div 
        class="fixed inset-0 bg-black/80 backdrop-blur-sm z-50 flex items-center justify-center"
        onclick={closeImage}
        onkeydown={(e) => e.key === 'Escape' && closeImage()}
        tabindex="0"
        role="dialog"
        aria-modal="true"
      >
        <div 
          class="max-w-[90vw] max-h-[90vh] relative"
          onclick={(e) => e.stopPropagation()}
        >
          <img 
            src={(selectedImage.image && selectedImage.image.display.url) || 
                (selectedImage.attachment && selectedImage.attachment.url) || 
                'https://via.placeholder.com/300'} 
            alt={selectedImage.title || 'Arena Block'} 
            class="max-w-full max-h-[90vh] object-contain"
          />
          {#if selectedImage.title}
            <div class="absolute bottom-0 left-0 right-0 bg-black/50 text-white p-2 text-sm">
              {selectedImage.title}
            </div>
          {/if}
        </div>
      </div>
    {/if}
  </div>
</div>

<style>
  :global(body) {
    background-color: black;
    margin: 0;
    padding: 0;
    overflow: hidden;
  }
  
  /* Disable touch callout on images to prevent context menus */
  :global(img) {
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    user-select: none;
  }

  /* Add smooth transition for modal */
  :global(.backdrop-blur-sm) {
    transition: backdrop-filter 0.2s ease-out;
  }
</style>