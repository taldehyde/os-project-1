# Nachos Project1 Report 
## b99901131 卓旻科

### Why the result is not congruent with expected?

因為兩個thread理論上應該要使用不同的pages，但原本的程式碼沒有做這樣的設定，導致兩個thread共用到相同的pages，所以程式就理所當然地出錯了。

###How do you modify the code?
	
原本程式執行./nachos -e test1 -e test2的結果如下：
		
	Total threads number is 2
	Thread ./test/test1 is executing.
	Thread ./test/test2 is executing.
	Print integer:9
	Print integer:8
	Print integer:7
	Print integer:20
	Print integer:21
	Print integer:22
	Print integer:23
	Print integer:24
	Print integer:6
	Print integer:7
	Print integer:8
	Print integer:9
	Print integer:10
	Print integer:12
	Print integer:13
	Print integer:14
	Print integer:15
	Print integer:16
	Print integer:16
	Print integer:17
	Print integer:18
	Print integer:19
	Print integer:20
	Print integer:17
	Print integer:18
	Print integer:19
	Print integer:20
	Print integer:21
	Print integer:21
	Print integer:23
	Print integer:24
	Print integer:25
	return value:0
	Print integer:26
	return value:0
	No threads ready or runnable, and no pending interrupts.
	Assuming the program completed.
	Machine halting!

	Ticks: total 800, idle 67, system 120, user 613
	Disk I/O: reads 0, writes 0
	Console I/O: reads 0, writes 0
	Paging: faults 0
	Network I/O: packets received 0, sent 0

建立一個static bool array來記錄哪些physical pages已經被使用

AddrSpace.h

    class AddrSpace {
    public:
		...
    private:
		...
    	static bool usedPhysPages[]; // an array records used physical pages
    };

在AddrSpace.cc裡將array初始化

	bool AddrSpace::usedPhysPages[NumPhysPages] = {FALSE};

原本的程式碼是將page table的生成放在constructor裡，現在我將它放到AddrSpace::Load()裡面，並且用算出來所需的page數量來生成page table，而不是像之前一樣直接使用最大值(NumPhysPages)。
而如果physical page已經被使用，則會找下一個未被使用的physical page，並將它assign給特定的virtual page。

in AddrSpace::Load(char *fileName)

    pageTable = new TranslationEntry[numPages];
    for (unsigned int i = 0, j = 0; i < numPages; i++) {
		pageTable[i].virtualPage = i;
        while(TRUE){
            if(usedPhysPages[j] == TRUE)
                {j++; continue;}
            break;
        }
        ASSERT(j < NumPhysPages);
		pageTable[i].physicalPage = j;
		usedPhysPages[j] = TRUE;
		//	pageTable[i].physicalPage = 0;
		pageTable[i].valid = TRUE;
		//	pageTable[i].valid = FALSE;
		pageTable[i].use = FALSE;
		pageTable[i].dirty = FALSE;
		pageTable[i].readOnly = FALSE;  
    }

之後將指令複製到memory上，physical page的address要用pageTable來換算

in AddrSpace::Load(char *fileName)

	if (noffH.code.size > 0) {
        DEBUG(dbgAddr, "Initializing code segment.");
		DEBUG(dbgAddr, noffH.code.virtualAddr << ", " << noffH.code.size);

        unsigned int vAddr = noffH.code.virtualAddr;
        unsigned int pAddr = pageTable[vAddr/PageSize].physicalPage * PageSize + vAddr%PageSize;

        	executable->ReadAt(
		&(kernel->machine->mainMemory[pAddr]),
			noffH.code.size, noffH.code.inFileAddr);
    }

在destructor將使用完的page歸還回去

in AddrSpace::~AddrSpace()

    for(unsigned int i = 0; i < NumPhysPages; i++){
        usedPhysPages[pageTable[i].physicalPage] = FALSE;
    }
    delete[] pageTable;

做了這些改動後，執行./nachos -e test1 -e test2的結果如下：

    Total threads number is 2
    Thread ./test/test1 is executing.
    Thread ./test/test2 is executing.
    Print integer:9
    Print integer:8
    Print integer:7
    Print integer:20
    Print integer:21
    Print integer:22
    Print integer:23
    Print integer:24
    Print integer:6
    return value:0
    Print integer:25
    return value:0
    No threads ready or runnable, and no pending interrupts.
    Assuming the program completed.
    Machine halting!
    
    Ticks: total 300, idle 8, system 70, user 222
    Disk I/O: reads 0, writes 0
    Console I/O: reads 0, writes 0
    Paging: faults 0
    Network I/O: packets received 0, sent 0

程式正確的執行了，並且是用Round-Robin(Nachos預設的scheduling)的方式輪流執行。

但是這裡產生了一個問題，當執行多個thread時，會因為physical page的數量不夠，而導致crash，
因此隨著執行的thread變多，kernel應該要能夠增加physical page的數目，才不會導致崩潰。