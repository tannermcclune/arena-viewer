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
  
  // Function to fetch a single page of channel data
  async function fetchChannelPage(page: number) {
    const response = await fetch(`https://api.are.na/v2/channels/${channelSlug}/contents?page=${page}&per=100&direction=desc&sort=position&filter=Image`);
    
    if (!response.ok) {
      throw new Error(`Failed to fetch channel page ${page}: ${response.statusText}`);
    }
    
    return await response.json();
  }
  
  // Modified fetchChannelData to randomize on load
  async function fetchChannelData() {
    loading = true;
    error = null;
    contents = [];
    
    try {
      // Update URL with current state
      updateUrl();
      
      // First, get the initial page to determine total pages
      const firstPage = await fetchChannelPage(1);
      totalPages = firstPage.total_pages || 1;
      currentPage = 1;
      
      // Add contents from first page (no need to filter as API only returns images)
      contents = [...firstPage.contents];
      
      console.log(`Channel has ${totalPages} pages with ${contents.length} images on page 1`);
      
      // Fetch remaining pages if any
      if (totalPages > 1) {
        // Fetch pages sequentially to avoid rate limiting
        for (let page = 2; page <= totalPages; page++) {
          console.log(`Fetching page ${page} of ${totalPages}`);
          try {
            const pageData = await fetchChannelPage(page);
            console.log(`Received ${pageData.contents.length} images from page ${page}`);
            contents = [...contents, ...pageData.contents];
          } catch (pageError) {
            console.error(`Error fetching page ${page}:`, pageError);
            // Continue with other pages even if one fails
          }
        }
      }
      
      console.log(`Total images loaded: ${contents.length}`);
      
      // Randomize the contents after loading all pages
      randomizeImages();
      
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
      console.log('New column count:', columnCount);
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
  <div class="flex-1 overflow-y-auto overflow-x-hidden p-0 relative">
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
            <img 
              src={(block.image && block.image.display.url) || 
                  (block.attachment && block.attachment.url) || 
                  'https://via.placeholder.com/300'} 
              alt={block.title || 'Arena Block'} 
              class="w-full h-full object-cover"
              loading="lazy"
            />
          </div>
        {/each}
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
</style>