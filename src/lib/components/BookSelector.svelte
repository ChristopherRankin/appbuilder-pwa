<!--
@component
The navbar component.
-->
<script>
    import Dropdown from './Dropdown.svelte';
    import SelectGrid from './SelectGrid.svelte';
    import TabsMenu from './TabsMenu.svelte';
    import { refs, nextRef, s, t, convertStyle, userSettings } from '$lib/data/stores';
    import { addHistory } from '$lib/data/history';
    import { DropdownIcon } from '$lib/icons';
    import { catalog } from '$lib/data/catalog';
    import config from '$lib/data/config';
    import SelectList from './SelectList.svelte';

    $: book = $nextRef.book === '' ? $refs.book : $nextRef.book;
    $: chapter = $nextRef.chapter === '' ? $refs.chapter : $nextRef.chapter;

    const listView = $userSettings['book-selection'] === 'list';
    const showChapterSelector = config.mainFeatures['show-chapter-selector-after-book'];
    const showVerseSelector = $userSettings['verse-selection'];

    // Translated book, chapter, and verse tab labels
    $: b = $t.Selector_Book;
    $: c = $t.Selector_Chapter;
    $: v = $t.Selector_Verse;

    let bookSelector;
    $: label = config.bookCollections
        .find((x) => x.id === $refs.collection)
        .books.find((x) => x.id === book).name;

    function chapterCount(book) {
        let count = Object.keys(books.find((x) => x.bookCode === book).versesByChapters).length;
        return count;
    }

    function verseCount(chapter) {
        if (!chapter || chapter === 'i') {
            return 0;
        }
        let count = Object.keys(chapters[chapter]).length;
        return count;
    }
    /**
     * Pushes reference changes to nextRef. Pushes final change to default reference.
     */
    function navigateReference(e) {
        if (!showChapterSelector) {
            $nextRef.book = e.detail.text;
            $refs = { book: $nextRef.book, chapter: 1 };
            document.activeElement.blur();
        } else {
            switch (e.detail.tab) {
                case b:
                    bookSelector.setActive(c);
                    $nextRef.book = e.detail.text;
                    if (chapterCount($nextRef.book) === 0) {
                        $nextRef.chapter = 'i';
                        completeNavigation();
                    }
                    break;
                case c:
                    $nextRef.chapter = e.detail.text;
                    if (verseCount($nextRef.chapter) === 0 || !showVerseSelector) {
                        completeNavigation();
                    } else {
                        bookSelector.setActive(v);
                    }
                    break;
                case v:
                    $nextRef.verse = e.detail.text;
                    completeNavigation();
                    break;
                default:
                    console.log('Book navigateReference: Default');
                    break;
            }
        }
    }

    async function completeNavigation() {
        $refs = { book: $nextRef.book, chapter: $nextRef.chapter };
        addHistory({
            collection: $refs.collection,
            book: $nextRef.book,
            chapter: $nextRef.chapter,
            verse: $nextRef.verse
        });
        document.activeElement.blur();
    }

    function resetNavigation() {
        if (bookSelector) {
            bookSelector.setActive(b);
        }
        nextRef.reset();
    }

    /**list of books in current docSet*/
    $: books = catalog.find((d) => d.id === $refs.docSet).documents;
    /**list of chapters in current book*/
    $: chapters = books.find((d) => d.bookCode === book).versesByChapters;

    let bookGridGroup = ({ colId, bookLabel = 'abbreviation' }) => {
        let groups = [];
        var lastGroup = null;

        config.bookCollections
            .find((x) => x.id === colId)
            .books.forEach((book) => {
                let label = book[bookLabel] || book.name;
                let cell = { label: label, id: book.id };
                let group = book.testament || '';
                if (lastGroup == null || group !== lastGroup) {
                    // Create new group
                    groups.push({
                        header: book.testament
                            ? $t['Book_Group_' + book.testament]
                            : lastGroup == null
                            ? '' // use empty string so that first group doesn't have header (e.g. INT)
                            : '\u00A0', // use &nbsp; so we have a blank space for additional books at the end
                        cells: [cell]
                    });
                    lastGroup = group;
                } else {
                    // Add Book to last group
                    let cells = groups[groups.length - 1].cells;
                    groups[groups.length - 1].cells = [...cells, cell];
                }
            });

        return groups;
    };

    let chapterGridGroup = (chapters) => {
        let hasIntroduction = books.find((x) => x.bookCode === book).hasIntroduction;
        return [
            {
                rows: hasIntroduction
                    ? [{ label: $t['Chapter_Introduction_Title'], id: 'i' }]
                    : null,
                cells: Object.keys(chapters).map((x) => ({ label: x, id: x }))
            }
        ];
    };
</script>

<!-- Book Selector -->
<Dropdown on:nav-end={resetNavigation}>
    <svelte:fragment slot="label">
        <div class="normal-case whitespace-nowrap" style={convertStyle($s['ui.selector.book'])}>
            {label}
        </div>
        <DropdownIcon color="white" />
    </svelte:fragment>
    <svelte:fragment slot="content">
        <div>
            <TabsMenu
                bind:this={bookSelector}
                options={{
                    [b]: {
                        component: listView ? SelectList : SelectGrid,
                        props: {
                            options: bookGridGroup({
                                colId: $refs.collection,
                                bookLabel: listView ? 'name' : 'abbreviation'
                            })
                        },
                        visible: true
                    },
                    [c]: {
                        component: SelectGrid,
                        props: {
                            options: chapterGridGroup(chapters)
                        },
                        visible: showChapterSelector
                    },
                    [v]: {
                        component: SelectGrid,
                        props: {
                            options: [
                                {
                                    cells: Object.keys(chapters[chapter]).map((x) => ({
                                        label: x,
                                        id: x
                                    }))
                                }
                            ]
                        },
                        visible: showChapterSelector && showVerseSelector
                    }
                }}
                on:menuaction={navigateReference}
            />
        </div>
    </svelte:fragment>
</Dropdown>
