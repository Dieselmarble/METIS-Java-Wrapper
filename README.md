### The JNA Java Wrapper for METIS - The Serial Graph Partitioning Tool

1. Introduction to METIS Graph Segmentation

METIS is a powerful graph segmentation software package developed by Karypis Lab. To be precise, METIS is a serial graph segmentation software package. Karypis Lab also provides a parallel version of the graph segmentation software package parMETIS and hMETIS that supports hypergraph and circuit division. The algorithm design of METIS is mainly based on the multi-level recursive dichotomy segmentation method, the multi-level K-way segmentation method and the multi-constraint partition mechanism. When users use the METIS software package, they can choose the corresponding segmentation method according to their needs.

The working principle of METIS: Take k-way multi-level division as an example.

The whole partition process is divided into three parts: coarsening, initial partitioning and uncoarsening (refinement). coarsening gradually reduces the size of the graph, G1->G2->G3-G4. K-way division is performed in the G4 phase, and then the original nodes in the graph are mapped to the clusters divided by G4 in the uncoarsening phase.



METIS installation: users need to download the latest METIS installation package from the official website of Karypis Lab, the latest version is METIS 5.1, and then decompress the package. It is necessary to ensure that the C compiler, GNU make and C Make 2.8 have been installed on the system, modify the value of the IDXTYPEWIDTH constant in the metis.h file to 32 or 64 according to the number of digits of the operating system, and then execute the make command under the metis underlying file. Both can complete the installation.



2. The use of METIS: 
Take gpmetis as an example, the usage method is gpmetis [options] graphfile nparts. gpmetis is the executable file generated by compilation, [options] is the option to execute gpmetis, graphfile is the file name to be divided, and nparts is the number of divided clusters specified by the user. The user can specify the gpmetis segmentation method by configuring the -ptype parameter of options. When -ptype = rb, the multi-level recursive binary segmentation algorithm is used. When -ptype = kway, the multi-level k-way segmentation algorithm is used (default value) . -ctype specifies the strategy for the coarsen operation. When -ctype = rm, random matching is used. When -ctype = shem, SHEM (Sorted heavy-edge matching) method is used for matching (default value).



- Format of METIS input and output files: The input file format is shown in the figure below.

The left image is an unweighted graph, and the first line is the number of vertices and the number of edges. Except for the first row, the i-th row indicates the vertex numbers connected by i-1 nodes. For example, line 2 indicates that there is an edge directly connected between vertex 1 and vertex 5, 3, and 2. The image on the right is a weighted graph, the first line indicates the number of vertices and the number of edges, and the format is a weighted graph. The i line indicates the vertex number connected by the i-1 node, followed by the weight value of the edge.



3. METIS compilation
Because the MAKE compilation on the personal computer was unsuccessful, I used a branch version on GitHub (see attachment), and then compiled it with VS2019. Just follow the normal compilation process. The packaged project is also in the attachment, you can click to open it directly.
The compilation options have been configured, remember to select DLL generation.

4. Self-made JNA interface
The self-made METIS JNA interface pays attention to the conversion of internal data types, uses malloc to create variable-length arrays, and then casts them into int64_t required by METIS. All the exchange codes with the JAVA interface are in JnaInterface.cpp. If you need to encapsulate other METS methods in the future, just continue to add them in this file. The corresponding JNA version package is also shown in the attachment.
the

A few points to note:

METIS has built the data format idx_t of the array itself, which is essentially int32 and in64, corresponding to different system versions, and the default is 64 bits. If you need to change, just modify IDXTYPEWIDTH=32/64 in metis.h. For interface functions interacting with JAVA, Char is passed by pointer, and both int and int[] arrays are passed by native arrays. After the transfer is completed, use malloc in C language to build a memory space, and then use the pointer of the address to call the real METIS source code method. Examples are as follows:


After building the memory space, manually assign a value to the memory space, and assign the native array passed in from Java to the memory space.

5. Java side code
On the Java side, first import the JNA plus package, and use the Native method in the figure below to load our self-compiled METIS.DLL.

After loading, we can directly use the interface functions written in the DLL.

6. Compressed Storage Format (CSR) stores adjacency matrix METIS uses CSR matrix to transfer graph structure information. The advantage of this is that using four simple int arrays, the entire graph structure can be represented. The first xadj is an array of pointers. In the two arrays in the code below, which point does the number to number belong to? The second adjncy is an adjacency array, which records the adjacency points of each point, and the truncation judgment can be made through xadj. The third is adjwgt is the weight of the edge. The fourth vwgt is the weight of the point.

The adjacency matrix of the graph structure is usually large and sparse, and the storage in the CSR format can save memory.

For each vertex i, its adjacency list is stored in consecutive locations in the array adjncy, and the array xadj is used to point to where it begins and where it ends. Figure 3(b) illustrates the CSR format for the 15-vertex graph shown in Figure 3(a).

Adjncy stores the neighbor point information of each vertex, and xadj stores the start and end positions of each vertex in xadj. For example: vertex 0 is adjacent to 1 and 5, in xadj, the starting position is 0, and the ending position is 2-1=1. vertex 1 is adjacent to 0, 2, and 6. In xadj, the starting position is 2 and the ending position is 5-1=4.

7. Appendix
The source code of all interface functions is as follows:

```
#include<stdio.h>
#include<stdlib.h>//要使用malloc是要包含此头文件
#include <memory.h>
#include "metislib.h"
 
int METIS_Interface(char *method, int contig, int nVerticesIn, int numEdgesIn, int nPartsIn, int nConIn, int ufactorIn,
int xadjIn[], int xadjncyIn[], int adjwgtIn[], int vwgtIn[], double tpwgtsIn[], int partAns[])
{
 
    idx_t nVertices = (idx_t)nVerticesIn;
    idx_t xadjLength = nVertices + 1;
    idx_t xadjncyLength = (idx_t)numEdgesIn * 2;
    idx_t objval;
    idx_t nParts = (idx_t)nPartsIn;
    idx_t nCon = (idx_t)nConIn; // nWeights
 
    idx_t i;
 
    //idx_t* part = imalloc(nVertices, "main: part");
    idx_t* part = (idx_t*)malloc(nVertices * sizeof(idx_t));
    idx_t* xadj = (idx_t*)malloc(xadjLength * sizeof(idx_t));
    idx_t* adjncy = (idx_t*)malloc(xadjncyLength * sizeof(idx_t));
    idx_t* adjwgt = (idx_t*)malloc(xadjncyLength * sizeof(idx_t));
    idx_t* vwgt = (idx_t*)malloc(nVertices * sizeof(idx_t));
    real_t* tpwgts = (real_t*)malloc(nParts * nCon * sizeof(real_t));
 
    //fill(adjncy, adjncy + xadjncyLength, 0);
 
    // Indexes of starting points in adjacent array
    for (i = 0; i < xadjLength; i++) {
        xadj[i] = (idx_t)xadjIn[i];
    }
 
    // Adjacent vertices in consecutive index order
    for (i = 0; i < xadjncyLength; i++) {
        adjncy[i] = (idx_t)xadjncyIn[i];
    }
 
 
    // Weights of vertices
    for (i = 0; i < nVertices; i++) {
        vwgt[i] = (idx_t)vwgtIn[i];
    }
 
    for (i = 0; i < xadjncyLength; i++) {
        adjwgt[i] = (idx_t)adjwgtIn[i];
    }
 
    // Target partition weight
    for (i = 0; i < nParts * nCon; i++) {
        tpwgts[i] = (real_t)tpwgtsIn[i];
    }
 
    // Initialise partition
    for (i = 0; i < nVertices; i++) {
        part[i] = 0;
    }
 
    idx_t options[METIS_NOPTIONS];
    METIS_SetDefaultOptions(options);
 
    // options[METIS_OPTION_CCORDER] = 1; // Sepecifies if the connected components of the graph should be identified and ordered separately
 
    options[METIS_OPTION_UFACTOR] = (idx_t)ufactorIn;
 
    options[METIS_OPTION_CONTIG] = (idx_t)contig; // enforce contiuous partition
 
    int ret;
    switch (method[0])
    {
    case 'k':
        ret = METIS_PartGraphKway(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, tpwgts, NULL, &options, &objval, part);
        break;
    case 'r':
        ret = METIS_PartGraphRecursive(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, tpwgts, NULL, &options, &objval, part);
        break;
    default:
        ret = METIS_PartGraphKway(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, tpwgts, NULL, &options, &objval, part);
    }
 
 
    //partition results to return, convert back to int[]
    for (i = 0; i < nVertices; i++) {
        partAns[i] = (int)part[i];
    }
 
    free(xadj);
    free(adjncy);
    free(adjwgt);
    free(part);
    free(vwgt);
    free(tpwgts);
 
    return ret;
}
 
 
int METIS_Interface_LayerTwo(char* method, int contig, int nVerticesIn, int numEdgesIn, int nPartsIn, int nConIn, int ufactorIn,
    int xadjIn[], int* xadjncyIn, int adjwgtIn[], int vwgtIn[], int partAns[]) {
 
    idx_t nVertices = (idx_t)nVerticesIn;
    idx_t xadjLength = nVertices + 1;
    idx_t xadjncyLength = (idx_t)numEdgesIn * 2;
    idx_t objval;
    idx_t nParts = (idx_t)nPartsIn;
    idx_t nCon = (idx_t)nConIn; // nWeights
 
    idx_t i;
 
    //idx_t* part = imalloc(nVertices, "main: part");
    idx_t* part = (idx_t*)malloc(nVertices * sizeof(idx_t));
    idx_t* xadj = (idx_t*)malloc(xadjLength * sizeof(idx_t));
    idx_t* adjncy = (idx_t*)malloc(xadjncyLength * sizeof(idx_t));
    idx_t* adjwgt = (idx_t*)malloc(xadjncyLength * sizeof(idx_t));
    idx_t* vwgt = (idx_t*)malloc(nVertices * sizeof(idx_t));
 
    //fill(adjncy, adjncy + xadjncyLength, 0);
 
    // Indexes of starting points in adjacent array
    for (i = 0; i < xadjLength; i++) {
        xadj[i] = (idx_t)xadjIn[i];
    }
 
    // Adjacent vertices in consecutive index order
    for (i = 0; i < xadjncyLength; i++) {
        adjncy[i] = (idx_t)xadjncyIn[i];
    }
 
    // Weights of vertices
    for (i = 0; i < nVertices; i++) {
        vwgt[i] = (idx_t)vwgtIn[i];
    }
 
    for (i = 0; i < xadjncyLength; i++) {
        adjwgt[i] = (idx_t)adjwgtIn[i];
    }
 
    // Initialise partition
    for (i = 0; i < nVertices; i++) {
        part[i] = 0;
    }
 
    idx_t options[METIS_NOPTIONS];
    METIS_SetDefaultOptions(options);
 
    // options[METIS_OPTION_CCORDER] = 1; // Sepecifies if the connected components of the graph should be identified and ordered separately
 
    options[METIS_OPTION_UFACTOR] = (idx_t)ufactorIn;
 
    options[METIS_OPTION_CONTIG] = (idx_t)contig; // enforce contiuous partition
 
    int ret;
    switch (method[0])
    {
    case 'k':
        ret = METIS_PartGraphKway(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, NULL, NULL, &options, &objval, part);
        break;
    case 'r':
        ret = METIS_PartGraphRecursive(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, NULL, NULL, &options, &objval, part);
        break;
    default:
        ret = METIS_PartGraphKway(&nVertices, &nCon, xadj, adjncy,
            vwgt, NULL, adjwgt, &nParts, NULL, NULL, &options, &objval, part);
    }
 
 
    //partition results to return, convert back to int[]
    for (i = 0; i < nVertices; i++) {
        partAns[i] = (int)part[i];
    }
 
    free(xadj);
    free(adjncy);
    free(adjwgt);
    free(part);
    free(vwgt);
 
    return ret;
}
```
