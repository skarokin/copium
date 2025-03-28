<script lang="ts">
    import { Input } from "$lib/components/ui/input";
    import { Separator } from "$lib/components/ui/separator";
    import * as Table from "$lib/components/ui/table/index.js";
    import * as HoverCard from "$lib/components/ui/hover-card/index.js";
    import * as AlertDialog from "$lib/components/ui/alert-dialog/index.js";
    import { Button } from "$lib/components/ui/button";
    import Switch from "$lib/components/ui/switch/switch.svelte";

    import {
        Map,
        Calendar,
        Building2,
        Link,
        BriefcaseBusiness,
        List,
        LayoutGrid,
    } from "lucide-svelte";

    import { formatDateForDisplay, formatDateForInput } from "$lib/utils/date";

    import { buildParamsFromFilters } from "$lib/utils/filter";
    import FilterPostings from "$lib/components/FilterPostings/FilterPostings.svelte";
    import { postingsFilterStore } from "$lib/stores/postingsFilterStore";

    import PaginatePostings from "$lib/components/PaginatePostings/PaginatePostings.svelte";
    import { postingsPaginationStore } from "$lib/stores/postingsPaginationStore";

    import type { PageData } from "./$types";
    import { onMount, onDestroy } from "svelte";
    import { goto } from "$app/navigation";
    import { browser } from "$app/environment";

    export let data: PageData;

    function addApplication(posting: any) {
        const company = posting.company_name;
        const role = posting.title;
        const location = posting.locations[0];
        const link = posting.url;
        const appliedDate = formatDateForInput(Math.floor(Date.now() / 1000));

        const form = new FormData();
        form.append("company", company);
        form.append("role", role);
        form.append("location", location);
        form.append("link", link);
        form.append("appliedDate", appliedDate.toString());

        // dont need to wait for response just fire and forget
        fetch("dashboard?/add", {
            method: "POST",
            body: form,
        });
    }

    function updateInput(e: Event) {
        const value = (e.currentTarget as HTMLInputElement).value;
        postingsFilterStore.update((current) => ({ ...current, query: value }));
    }

    // only if input is focused, handle Enter
    function handleKeyDown(e: KeyboardEvent) {
        if (e.key === "Enter") {
            const queryElement = document.getElementById(
                "query"
            ) as HTMLInputElement | null;
            if (document.activeElement !== queryElement) return;
            e.preventDefault();

            // update the store with the new query
            updateInput(e);
            updateURL(); // updateURL will use raw text query AND filter values from FilterJobs component
        }
    }

    // shortcuts for search input; only works when body is focused
    function handleGlobalKeydown(e: KeyboardEvent) {
        if (e.key == "Escape") {
            const queryElement = document.getElementById(
                "query"
            ) as HTMLInputElement | null;
            if (queryElement) queryElement.blur();
        }
    }

    function updateURL() {
        const { query, company, role, location, startDate, endDate } =
            $postingsFilterStore;
        const params = buildParamsFromFilters({
            query,
            company,
            role,
            location,
            startDate,
            endDate,
            // we can load more in grid view
            hitsPerPage: isGridView ? 20 : 10,
        });

        // this has to be updated in updateURL as well because user may have changed view preference
        postingsPaginationStore.update((current) => ({
            ...current,
            count: isGridView ? 20 * data.totalPages : 10 * data.totalPages,
            perPage: isGridView ? 20 : 10,
        }));

        goto(`?${params.toString()}`);
    }

    // list - grid style toggle
    let isViewPreferenceLoaded = false;
    let isGridView: boolean | undefined = undefined;

    // reactive block to update pagination count
    //  - onMount does not work here since page data is updated but
    //    component is not rerendered when goto() is called
    $: postingsPaginationStore.update((current) => ({
        ...current,
        count: isGridView ? 20 * data.totalPages : 10 * data.totalPages,
        perPage: isGridView ? 20 : 10,
    }));

    // update view preference as soon as possible
    $: if (browser && isGridView === undefined) {
        const savedView = localStorage.getItem("view_preference_postings");
        isGridView = savedView === "true";
        isViewPreferenceLoaded = true;
    }

    // save view preference
    $: if (browser && isViewPreferenceLoaded && isGridView !== undefined) {
        localStorage.setItem("view_preference_postings", isGridView.toString());
        updateURL(); // must reload with saved view preference because of hitsPerPage
    }

    onMount(() => {
        window.addEventListener("keydown", handleGlobalKeydown);
        
        if (browser && window.innerWidth < 640) {
            isGridView = true;
        }

        return () => {
            window.removeEventListener("keydown", handleGlobalKeydown);
        };
    });

    onDestroy(() => {
        if (typeof window !== "undefined") {
            window.removeEventListener("keydown", handleGlobalKeydown);
        }
    });

</script>

<div class="flex flex-col justify-start gap-4 items-stretch w-full my-12">
    <!-- sticky controls wrapper -->
    <div
        class="px-6 sm:px-8 sticky bg-background/95 backdrop-blur supports-[backdrop-filter]:bg-background/60 z-50"
    >
        <div
            class="bg-background flex flex-col sm:flex-row justify-between gap-2 sm:gap-4 items-center w-full py-2"
        >
            <div class="flex flex-col w-full">
                <div
                    class="flex flex-row gap-4 items-center w-full sm:w-auto mb-4"
                >
                    <Input
                        type="text"
                        placeholder="Search by company, role, or location."
                        id="query"
                        on:input={updateInput}
                        on:keydown={handleKeyDown}
                    />
                    <Separator orientation="vertical" class="h-6" />
                    <FilterPostings />
                </div>
                {#if isViewPreferenceLoaded}
                    <div
                        class="flex flex-row gap-4 justify-center sm:justify-between items-center w-full sm:w-auto"
                    >
                        <div class="hidden sm:flex flex gap-2 items-center justify-center">
                            <div
                                class={!isGridView
                                    ? "flex items-center gap-1 text-sm font-medium"
                                    : "flex items-center gap-1 text-muted-foreground text-sm font-medium"}
                            >
                                <List class="w-[15px] h-[17px] stroke-[1.5]" />
                                List
                            </div>
                            <Switch bind:checked={isGridView} />
                            <div
                                class={isGridView
                                    ? "flex items-center gap-1 text-sm font-medium"
                                    : "flex items-center gap-1 text-muted-foreground text-sm font-medium"}
                            >
                                <LayoutGrid
                                    class="w-[15px] h-[17px] stroke-[1.5]"
                                />
                                Grid
                            </div>
                        </div>
                        <PaginatePostings />
                    </div>
                {/if}
            </div>
        </div>
    </div>

    {#if isViewPreferenceLoaded}
        {#if !isGridView}
            <Table.Root class="overflow-collapse table-fixed">
                <Table.Header>
                    <Table.Row class="border-b border-dashed">
                        <Table.Head
                            class="border-r border-dashed pl-3 lg:pl-6 w-[1px]"
                        ></Table.Head>
                        <Table.Head class="border-r border-dashed lg:pl-4 2xl:pl-6 w-3/12">
                            <span class="inline-flex items-center gap-2">
                                <Building2
                                    class="w-[15px] h-[17px] stroke-[1.5]"
                                />
                                Company
                            </span>
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed w-5/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                            <span class="inline-flex items-center gap-2">
                                <BriefcaseBusiness
                                    class="w-[15px] h-[17px] stroke-[1.5]"
                                />
                                Role
                            </span>
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed w-1/6 md:w-1/6 px-2 lg:pl-4 2xl:pl-6"
                        >
                            <span class="inline-flex items-center gap-2">
                                <Map class="w-[15px] h-[17px] stroke-[1.5]" />
                                Locations
                            </span>
                        </Table.Head>
                        <Table.Head
                            class="hidden lg:table-cell border-r border-dashed w-2/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                            <span class="inline-flex items-center gap-2">
                                <Calendar
                                    class="w-[15px] h-[17px] stroke-[1.5]"
                                />
                                Posted
                            </span>
                        </Table.Head>
                        <Table.Head
                            class="hidden lg:table-cell border-r border-dashed w-2/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                            <span class="inline-flex items-center gap-2">
                                <Calendar
                                    class="w-[15px] h-[17px] stroke-[1.5]"
                                />
                                Updated
                            </span>
                        </Table.Head>
                        <Table.Head class="w-1/6 md:w-1/12 border-r border-dashed">
                            <span class="w-full inline-flex items-center justify-center gap-2">
                                <Link class="w-[15px] h-[17px] stroke-[1.5]" />
                                Link
                            </span>
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed pl-3 lg:pl-6 w-[1px]"
                        ></Table.Head>
                    </Table.Row>
                </Table.Header>
                <Table.Body>
                    {#each data.postings as posting, i (i)}
                        <Table.Row
                            class="w-border-b border-dashed dark:brightness-[0.9]"
                        >
                            <Table.Cell class="border-r border-dashed w-[1px]"
                            ></Table.Cell>
                            <Table.Cell
                                class="border-r border-dashed w-3/12 px-2 lg:pl-4 2xl:pl-6"
                            >
                                <div class="flex items-center gap-2">
                                    {#if data.companyLogos[posting.company_name]}
                                        <img
                                            src={data.companyLogos[posting.company_name]}
                                            alt={posting.company_name}
                                            class="w-6 h-6 rounded-lg object-cover"
                                        />
                                    {:else}
                                        <div class="w-6 h-6 p-1 rounded-lg flex items-center justify-center border border-zinc-400 border-opacity-50 dark:border-opacity-40">
                                            <BriefcaseBusiness class="stroke-[1.5] text-zinc-400 opacity-70 dark:opacity-50" />
                                        </div>
                                    {/if}
                                    <p class="truncate">
                                        {posting.company_name}
                                    </p>
                                </div>
                            </Table.Cell>
                            <Table.Cell class="px-0 border-r border-dashed">
                                <p class="truncate px-2 lg:pl-4 2xl:pl-6">
                                    {posting.title}
                                </p>
                            </Table.Cell>
                            <Table.Cell class="px-0 border-r border-dashed">
                                <HoverCard.Root>
                                    <HoverCard.Trigger
                                        class="rounded-sm underline-offset-4 hover:underline focus-visible:outline-2 focus-visible:outline-offset-8 focus-visible:outline-black"
                                    >
                                        <p class="truncate px-2 lg:pl-4 2xl:pl-6">
                                            {#if posting.locations.length > 1}
                                                {posting.locations[0]}
                                                <span class="font-semibold"
                                                    >+{posting.locations
                                                        .length - 1}</span
                                                >
                                            {:else}
                                                {posting.locations[0]}
                                            {/if}
                                        </p>
                                    </HoverCard.Trigger>
                                    <HoverCard.Content class="w-fit">
                                        <div
                                            class="flex justify-between space-x-4"
                                        >
                                            <div class="space-y-1">
                                                {#each posting.locations as location}
                                                    <p class="text-sm">
                                                        {location}
                                                    </p>
                                                {/each}
                                            </div>
                                        </div>
                                    </HoverCard.Content>
                                </HoverCard.Root>
                            </Table.Cell>
                            <Table.Cell class="hidden lg:table-cell px-0 border-r border-dashed"
                                ><p class="truncate px-2 lg:pl-4 2xl:pl-6">
                                    {formatDateForDisplay(posting.date_posted)}
                                </p></Table.Cell
                            >
                            <Table.Cell class="hidden lg:table-cell px-0 border-r border-dashed"
                                ><p class="truncate px-2 lg:pl-4 2xl:pl-6">
                                    {formatDateForDisplay(posting.date_updated)}
                                </p></Table.Cell
                            >
                            <Table.Cell
                                class="flex items-center justify-center w-full"
                            >
                                <AlertDialog.Root>
                                    <AlertDialog.Trigger asChild let:builder>
                                        <Button
                                            builders={[builder]}
                                            href={posting.url}
                                            target="_blank"
                                            size="sm"
                                        >
                                            Apply
                                        </Button>
                                    </AlertDialog.Trigger>
                                    <AlertDialog.Content>
                                        <AlertDialog.Header>
                                            <AlertDialog.Title>
                                                Did you apply for this job?
                                            </AlertDialog.Title>
                                            <AlertDialog.Description>
                                                Click "Yes" below to
                                                automatically add this to your
                                                dashboard.
                                            </AlertDialog.Description>
                                        </AlertDialog.Header>
                                        <AlertDialog.Footer>
                                            <AlertDialog.Cancel
                                                >No</AlertDialog.Cancel
                                            >
                                            <AlertDialog.Action
                                                on:click={() =>
                                                    addApplication(posting)}
                                            >
                                                Yes
                                            </AlertDialog.Action>
                                        </AlertDialog.Footer>
                                    </AlertDialog.Content>
                                </AlertDialog.Root>
                            </Table.Cell>
                            <Table.Cell class="border-l border-dashed w-[0.1px]"
                            ></Table.Cell>
                        </Table.Row>
                    {/each}
                </Table.Body>
                <Table.Footer class="border-none bg-background">
                    <Table.Row class="border-t border-dashed border-b-0 hover:bg-background">
                        <Table.Head
                            class="border-r border-dashed pl-3 lg:pl-6 w-[1px]"
                        ></Table.Head>
                        <Table.Head class="border-r border-dashed lg:pl-4 2xl:pl-6 w-3/12">
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed w-5/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed w-1/6 md:w-1/6 px-2 lg:pl-4 2xl:pl-6"
                        >
                        </Table.Head>
                        <Table.Head
                            class="hidden lg:table-cell border-r border-dashed w-2/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                        </Table.Head>
                        <Table.Head
                            class="hidden lg:table-cell border-r border-dashed w-2/12 px-2 lg:pl-4 2xl:pl-6"
                        >
                        </Table.Head>
                        <Table.Head class="w-1/6 md:w-1/12 border-r border-dashed">
                        </Table.Head>
                        <Table.Head
                            class="border-r border-dashed pl-3 lg:pl-6 w-[1px]"
                        ></Table.Head>
                    </Table.Row>
                </Table.Footer>
            </Table.Root>
        {:else}
            <!-- grid view -->
            <div
                class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 xl:grid-cols-5 gap-4 px-6 sm:px-8"
            >
                {#each data.postings as posting, i (i)}
                    <div
                        class="border rounded-lg p-4 flex flex-col gap-1 h-full dark:brightness-[0.9]"
                    >
                        <div class="flex items-center gap-2 mb-2">
                            {#if data.companyLogos[posting.company_name]}
                                        <img
                                            src={data.companyLogos[posting.company_name]}
                                            alt={posting.company_name}
                                            class="w-8 h-8 rounded-lg object-cover"
                                        />
                                    {:else}
                                    <div class="w-8 h-8 p-1 rounded-lg flex items-center justify-center border border-zinc-400 border-opacity-50 dark:border-opacity-40">
                                        <BriefcaseBusiness class="stroke-[1.5] text-zinc-400 opacity-70 dark:opacity-50" />
                                    </div>
                                    {/if}
                            <h3 class="font-medium truncate flex-1">
                                {posting.company_name}
                            </h3>

                            <div class="mt-auto">
                                <AlertDialog.Root>
                                    <AlertDialog.Trigger asChild let:builder>
                                        <Button
                                            builders={[builder]}
                                            href={posting.url}
                                            target="_blank"
                                            size="sm"
                                        >
                                            Apply
                                        </Button>
                                    </AlertDialog.Trigger>
                                    <AlertDialog.Content>
                                        <AlertDialog.Header>
                                            <AlertDialog.Title>
                                                Did you apply for this job?
                                            </AlertDialog.Title>
                                            <AlertDialog.Description>
                                                Click "Yes" below to
                                                automatically add this to your
                                                dashboard.
                                            </AlertDialog.Description>
                                        </AlertDialog.Header>
                                        <AlertDialog.Footer>
                                            <AlertDialog.Cancel
                                                >No</AlertDialog.Cancel
                                            >
                                            <AlertDialog.Action
                                                on:click={() =>
                                                    addApplication(posting)}
                                            >
                                                Yes
                                            </AlertDialog.Action>
                                        </AlertDialog.Footer>
                                    </AlertDialog.Content>
                                </AlertDialog.Root>
                            </div>
                        </div>

                        <h4 class="text-sm font-medium truncate">
                            {posting.title}
                        </h4>

                        <div
                            class="text-xs text-muted-foreground flex flex-col items-start"
                        >
                            <HoverCard.Root>
                                <HoverCard.Trigger
                                    class="flex gap-1 items-center rounded-sm underline-offset-4 hover:underline focus-visible:outline-2 focus-visible:outline-offset-8 focus-visible:outline-black"
                                >
                                    <Map class="w-3 h-3 inline" />
                                    <p class="truncate">
                                        {#if posting.locations.length > 1}
                                            {posting.locations[0]}
                                            <span class="font-semibold"
                                                >+{posting.locations.length -
                                                    1}</span
                                            >
                                        {:else}
                                            {posting.locations[0]}
                                        {/if}
                                    </p>
                                </HoverCard.Trigger>
                                <HoverCard.Content class="w-fit">
                                    <div class="flex justify-between space-x-4">
                                        <div class="space-y-1">
                                            {#each posting.locations as location}
                                                <p class="text-sm">
                                                    {location}
                                                </p>
                                            {/each}
                                        </div>
                                    </div>
                                </HoverCard.Content>
                            </HoverCard.Root>
                            <div
                                class="text-xs text-muted-foreground flex items-center gap-1"
                            >
                                <Calendar class="w-3 h-3 inline" />
                                <span
                                    >Posted: {formatDateForDisplay(
                                        posting.date_posted
                                    )}</span
                                >
                            </div>
                        </div>
                    </div>
                {/each}
            </div>
        {/if}
    {/if}

    <div class="container flex flex-col md:flex-row justify-end md:gap-4">
        <p class="text-muted-foreground text-xs">*Only shows active postings</p>
        <!-- <p class="text-muted-foreground text-xs">*Updated every 3 hours</p> -->
        <div class="flex space-x-1">
            <p class="text-muted-foreground text-xs">
                *Internship postings from
            </p>
            <a
                href="https://github.com/cvrve/Summer2025-Internships"
                class="text-muted-foreground text-xs hover:text-foreground/50">[cvrve]</a
            >
        </div>
    </div>
</div>
