#include <Magick++.h>
#include <fstream.h>
#include <math.h>
#include <sys/time.h>
#include <unistd.h>
#include <iostream>
#include <pthread.h>
#include <stdio.h>
#include <math.h>
#include <time.h>

using namespace Magick;
using namespace std;
fstream fout;

typedef struct img_data{
  int **intensity;
  int col;
  int row;
}img_data;

typedef struct thread_data{
  int order;
  int **intensity;
  int **coef;
  int col;
  int row;
  char writecode[80*80*26*32];
  int length;
}thread_data;

void loadImg(img_data *ps_img,char* file);
void printImg(img_data pimg_data);
void divideImg(img_data pimg_data, thread_data *pth_data,int  num_threads);

void *pth_DCT_Quantz(void* pth_data);
void *pth_encode(void* pth_data);

void Quantize(int F[8][8], int QF[8][8]);
void DCT(int f[8][8],int F[8][8]);
int RLE(int ZZ[64],int RL[64]);

void writeToFile(char* a);


int main(int argc, char** argv)
{
  clock_t clk;
  
  //VARIABLE
  char* fname;
  char* fname_out;
  int num_threads;
  pthread_t *threadIDs;
  thread_data *pth_data;//array
  img_data pimg_data;
  int rc;
  int i,j,k;
  char writecode[80*80*26*32];

  //GET USER INPUTS
  fname = (char*)argv[1];
  sscanf(argv[2], "%d", &num_threads);
  fname_out = (char*) argv[3];
  //cout<<"\nFname"<<fname<<"Num threads"<<num_threads<<"Fname_out "<<fname_out;
  fout.open(fname_out,ios::out);
  
  //LOAD IMAGE
  loadImg(&pimg_data,fname);
  //printImg(pimg_data); 
  
  //INITIALIZATION
  threadIDs = new pthread_t[num_threads];
  pth_data = new thread_data[num_threads];
  
  
  clk = clock();
  divideImg(pimg_data, pth_data, num_threads);
  clock_t clk1 = clock();
  cout<<"\n Divide  Time eslaped: "<<(float)(clk1-clk)/CLOCKS_PER_SEC;
  for(i=0;i<num_threads;i++)
  {
    rc = pthread_create(&threadIDs[i], NULL, pth_DCT_Quantz, (void*) &pth_data[i]);
  }
  
  for(i=0;i<num_threads;i++)
  {
    void* status;
    rc = pthread_join(threadIDs[i],&status);
    if (rc) {
      cout<<"ERROR; return code from pthread_join()is" << rc;
      exit(-1);
    }else{
      cout<<"Main: completed join with thread"<<i<<" having a status  of"<<(long)status;
    cout<<"in time: "<<(float)(clock()-clk1)/CLOCKS_PER_SEC;
      }
  }
  clock_t clk2 = clock();
  cout<<"\n DC Time eslaped: "<<(float)(clk2-clk1)/CLOCKS_PER_SEC;
 
  //Subtract DC
  int lastDC = 0;
  //int count = 0;
  for(i = 0;i<num_threads;i++){
    for(j=0;j<pth_data[i].row;j=j+8)
      for(k=0;k<pth_data[i].col;k=k+8){
	//count++;
	//cout<<"\n Thread "<<i<<" DC "<<j<<"-"<<k<<"|"<<pth_data[i].row<<"x"<<pth_data[i].col<<":"<<pth_data[i].coef[j][k];
	int tempDC = pth_data[i].coef[j][k];
	pth_data[i].coef[j][k] = pth_data[i].coef[j][k]-lastDC;
	lastDC = tempDC;
	//cout<<"\nDC["<<count<<"]="<<lastDC;
       }
  }
  clock_t clk3 = clock();
   cout<<"\n DC Subtract  Time eslaped: "<<(float)(clk3-clk2)/CLOCKS_PER_SEC;
  //Do the rest of ecoding
  for(i=0;i<num_threads;i++)
  {
    rc = pthread_create(&threadIDs[i], NULL, pth_encode, (void*) &pth_data[i]);
  }
  for(i=0;i<num_threads;i++)
  {
    rc = pthread_join(threadIDs[i],NULL);
    }
  //combine code
  writecode[0] = '\0';
  for(int i=0;i<num_threads;i++)
  {
    strcat(writecode,pth_data[i].writecode);    
  }
  
  char *end = "111111111";
  strcat(writecode,end);
  clock_t clk4 = clock(); 
  char x = ((pimg_data.col)/8);
  char y = ((pimg_data.row)/8);
  fout<<x<<y;
  writeToFile(writecode);
  fout.close();

  cout<<"\n Rest encoding Time eslaped: "<<(float)(clk4-clk3)/CLOCKS_PER_SEC;
  return 0;
}

void loadImg(img_data *ps_img,char* file)
{
  cout<<"\nLoadImage: \n"<<file;
  int cRes = 255;
  int **intensity;
  Image image(file);
  int maxX=image.columns();
  int maxY=image.rows();
  cout<<"\nOriginal Size: "<<maxY<<"x"<<maxX;
  image.classType(DirectClass);
  Pixels view(image);
  PixelPacket *pixels=view.get(0,0,maxX,maxY);
  int i,j;
  //padding
  int row8 = ((maxY+7)/8)*8;
  int col8 = ((maxX+7)/8)*8;
  cout<<"\nPadding "<<row8<<"x"<<col8;
  intensity = new int*[row8];
  for(int i=0;i<row8;i++)
    intensity[i] = new int[col8];
  for(i=0;i<row8;i++)
    for(j=0;j<col8;j++)
      {
	if((i<maxY)&&(j<maxX))
	  {
	    cout.flush();
	    ColorGray gray=Color(*(pixels+i*maxX+j));
	    intensity[i][j]=(int)(gray.shade()*cRes);
	  }
	else
	  intensity[i][j] = 0;
      }
  ps_img->intensity = intensity;
  ps_img->col = col8;
  ps_img->row = row8;
  cout<<"\n Image size padded "<<row8<<"x"<<col8;
}


void *pth_DCT_Quantz(void* pth_data)
{
  //  fout.open(fname,ios::out);

  int i,j,i1,j1;
  int f[8][8];
  //int DCcomp = 0;
  //char code[1000];
  // char writecode[80*80*26*32]; 
  // writecode[0] = '\0';
  thread_data* my_data = (thread_data*)pth_data;
  cout<<"\nThread "<<(int)my_data->order;
  int maxX = my_data->col;//padded8
  int maxY = my_data->row;//padded8
  cout<<"\nThread Size "<<maxY<<"x"<<maxX;
  int **intensity = my_data->intensity;
  int **coef = new int*[maxY];
  for(int k = 0; k < maxY; k++)
    coef[k]= new int[maxX];

  int  x = (maxX/8);
  int  y = (maxY/8);
  //cout<<"\nAsset: "<<intensity[maxY-1][maxX-1];
  //cout<<"\nNumber of Blocks : "<<(int)y<<"*"<<(int)x<<"\n";
  for(i=0;i<maxY/8;i++)
    for(j=0;j<maxX/8;j++)
    {	
	for(i1=0;i1<8;i1++)
	  for(j1=0;j1<8;j1++)
	    {
	      //cout<<"\n pixel "<<i*8+i1<<"x"<<j*8+j1<<"|"<<maxY<<"x"<<maxX;
	      if((i*8+i1<maxY)&&(j*8+j1<maxX))
		f[i1][j1] = intensity[i*8+i1][j*8+j1]-128;
	      else
		f[i1][j1] = 0;
	    }
        //cout<<"\nProcessing Block : "<<i<<"*"<<j;
	//getch();
	int F[8][8];
	DCT(f, F);
	int QF[8][8];
	Quantize(F, QF);
	//cout<<" DC="<<QF[0][0];
	//cout<<"\nStart copying coeff";
	for(i1=0;i1<8;i1++)
	  for(j1=0;j1<8;j1++)
	  coef[i*8+i1][j*8+j1] = QF[i1][j1];
    }  
  my_data->coef = coef;
  //char *end = "111111111";
  //strcat(writecode,end);
  //writeToFile(writecode);
  //fout.close();
}

void printImg(img_data pimg_data)
{
  //for(int i=0;i<pimg_data.row;i++)
  //{
  //  for(int j=0;j<pimg_data.col;j++)
  //    cout<<" "<<pimg_data.intensity[i][j];
  //  cout<<"\n";
  //}
  cout<<"\n"<<"Image Size: "<<pimg_data.row<<"x"<<pimg_data.col;
}

void divideImg(img_data pimg_data, thread_data *pth_data, int num_threads)
{
  //num block per row
  int col = pimg_data.col;
  int nblk_row = pimg_data.row/8;//padded
  //num_block_per thread
  int nblk_thread = ceil((double)nblk_row/num_threads);
  cout<<"\nBlock per Row "<<nblk_row<<" Block per Thread "<<nblk_thread;
  for(int n = 0; n<num_threads; n++){
    pth_data[n].order = n;
    int start_row = n*nblk_thread*8;
    int end_row = start_row + nblk_thread*8-1;
    if(end_row > pimg_data.row){
      end_row = (nblk_row-1)* 8;
    }
    //cout<<"\nRow S-E: "<<start_row<<"-"<<end_row;
    int num_row = end_row - start_row + 1;
    pth_data[n].intensity = new int*[num_row];
    for(int k = 0; k< num_row; k++)
      pth_data[n].intensity[k] = new int[col];
    //copy data
    for(int i = 0; i<num_row;i++){
      for(int j = 0; j<pimg_data.col;j++)	
	pth_data[n].intensity[i][j] = pimg_data.intensity[start_row+i][j];
    }
    pth_data[n].col = pimg_data.col;
    pth_data[n].row = num_row;
  }
}



void Quantize(int F[8][8], int QF[8][8])
{

  int q[8][8] = {16,11,10,16,24,40,51,61,
		 12,12,14,19,26,58,60,55,
		 14,13,16,24,40,57,69,56,
		 14,17,22,29,51,87,80,62,
		 18,22,37,56,68,109,103,77,
		 24,35,55,64,81,104,113,92,
		 49,64,78,87,103,121,120,101,
		 72,92,95,98,112,100,103,99 };
  int i,j;
  for(i=0;i<8;i++)
    for(j=0;j<8;j++)
      QF[i][j] = int(F[i][j]/q[i][j]);
  //cout<<"\nQuantize done";
}
float C(int u)
{
  if(u==0)
    return (1.0/sqrt(2.0));
  else
    return 1.0;
}
void DCT(int f[8][8],int F[8][8])
{

  float a;
  for(int u=0;u<8;u++)
    for(int v=0;v<8;v++)
      {
	a = 0.0;
	for(int x=0;x<8;x++)
	  for(int y=0;y<8;y++)
	    a += float(f[x][y])*cos((2.0*float(x)+1.0)*float(u)*3.14/16.0)*cos((2.0*float(y)+1.0)*float(v)*3.14/16.0);
	F[u][v] = int(0.25*C(u)*C(v)*a);
      }
  //cout<<"\n DCT";
}

int RLE(int ZZ[64],int RL[64])
{
  int rl=1;
  int i=1;
  int k = 0;
  RL[0] = ZZ[0];
  while(i<64)
    {
      k=0;
      while((i<64)&&(ZZ[i]==0)&&(k<15))
	{
	  i++;
	  k++;
	}
      if(i==64)
	{
	  RL[rl++] = 0;
	  RL[rl++] = 0;
	}
      else
	{ 
	  RL[rl++] = k;
	  RL[rl++] = ZZ[i++];
	}
    }
  if(!(RL[rl-1]==0 && RL[rl-2]==0))
    {
      RL[rl++] = 0;
      RL[rl++] = 0;
    }
  if((RL[rl-4]==15)&&(RL[rl-3]==0))
    {
      RL[rl-4]=0;
      rl-=2;
    }
  return rl;
}
void writeToFile(char* a)
{
 int len = strlen(a);
 int b = 0;
 int c = len/8;
 int i,j;
 //cout<<"c : "<<c;
 char d;
 for(i=0;i<=c;i++)
 {
   b = 0;
   for(j=0;j<8;j++)
      if((i*8+j<len)&&(a[i*8+j]=='1'))
	b = b | (int)pow(2,7-j);
   d = b;
   fout<<d;
   //   cout<<b<<" ";
 }
}

int getCat(int a)
{
  if(a==0)
	return 0;
  else if(abs(a)<=1)
	return 1;
  else if(abs(a)<=3)
	return 2;
  else if(abs(a)<=7)
	return 3;
  else if(abs(a)<=15)
	return 4;
  else if(abs(a)<=31)
	return 5;
  else if(abs(a)<=63)
	return 6;
  else if(abs(a)<=127)
	return 7;
  else if(abs(a)<=255)
	return 8;
  else if(abs(a)<=511)
	return 9;
  else if(abs(a)<=1023)
	return 10;
  else if(abs(a)<=2047)
	return 11;
  else if(abs(a)<=4095)
	return 12;
  else if(abs(a)<=8191)
	return 13;
  else if(abs(a)<=16383)
	return 14;
  else
	return 15;
}

void getDCcode(int a,int& lenb,char *b)
{
  int codeLen[12] = {3,4,5,5,7,8,10,12,14,16,18,20};
  char* code[12] = {"010","011","100","00","101","110","1110","11110","111110","1111110","11111110","111111110"};
  int cat = getCat(a);
  lenb = codeLen[cat];
  strcpy(b,code[cat]);
  int j;
  int c = a;
  if(a<0)
     c+=(int)pow(2,cat)-1;
  for(j=lenb-1;j>lenb-cat-1;j--)
  {
     if(c%2==1)
	b[j] = '1';
     else
	b[j] = '0';
     c/=2;
  }
  b[lenb] = '\0';
}

void getACcode(int n,int a, int& lenb, char* b)
{
  int codeLen[16][11] = {
  4 ,3 ,4 ,6 ,8 ,10,12,14,18,25,26,
  0 ,5 ,8 ,10,13,16,22,23,24,25,26,
  0 ,6 ,10,13,20,21,22,23,24,25,26,
  0 ,7 ,11,14,20,21,22,23,24,25,26,
  0 ,7 ,12,19,20,21,22,23,24,25,26,
  0 ,8 ,12,19,20,21,22,23,24,25,26,
  0 ,8 ,13,19,20,21,22,23,24,25,26,
  0 ,9 ,13,19,20,21,22,23,24,25,26,
  0 ,9 ,17,19,20,21,22,23,24,25,26,
  0 ,10,18,19,20,21,22,23,24,25,26,
  0 ,10,18,19,20,21,22,23,24,25,26,
  0 ,10,18,19,20,21,22,23,24,25,26,
  0 ,11,18,19,20,21,22,23,24,25,26,
  0 ,12,18,19,20,21,22,23,24,25,26,
  0 ,13,18,19,20,21,22,23,24,25,26,
  12,17,18,19,20,21,22,23,24,25,26
  };
  char* code[16][11] = {
  "1010",  "00",  "01",  "100",  "1011",  "11010",  "111000",  "1111000",  "1111110110",  "1111111110000010",  "1111111110000011",
  "","1100","111001","1111001","111110110","11111110110","1111111110000100","1111111110000101","1111111110000110","1111111110000111","1111111110001000",
  "","11011","11111000","1111110111","1111111110001001","1111111110001010","1111111110001011","1111111110001100","1111111110001101","1111111110001110","1111111110001111",
  "","111010","111110111","11111110111","1111111110010000","1111111110010001","1111111110010010","1111111110010011","1111111110010100","1111111110010101","1111111110010110",
  "","111011","1111111000","1111111110010111","1111111110011000","1111111110011001","1111111110011010","1111111110011011","1111111110011100","1111111110011101","1111111110011110",
  "","1111010","1111111001","1111111110011111","1111111110100000","1111111110100001","1111111110100010","1111111110100011","1111111110100100","1111111110100101","1111111110100110",
  "","1111011","11111111000","1111111110100111","1111111110101000","1111111110101001","1111111110101010","1111111110101011","1111111110101100","1111111110101101","1111111110101110",
  "","11111001","11111111001","1111111110101111","1111111110110000","1111111110110001","1111111110110010","1111111110110011","1111111110110100","1111111110110101","1111111110110110",
  "","11111010","111111111000000","1111111110110111","1111111110111000","1111111110111001","1111111110111010","1111111110111011","1111111110111100","1111111110111101","1111111110111110",
  "","111111000","1111111110111111","1111111111000000","1111111111000001","1111111111000010","1111111111000011","1111111111000100","1111111111000101","1111111111000110","1111111111000111",
  "","111111001","1111111111001000","1111111111001001","1111111111001010","1111111111001011","1111111111001100","1111111111001101","1111111111001110","1111111111001111","1111111111010000",
  "","111111010","1111111111010001","1111111111010010","1111111111010011","1111111111010100","1111111111010101","1111111111010110","1111111111010111","1111111111011000","1111111111011001",
  "","1111111010","1111111111011010","1111111111011011","1111111111011100","1111111111011101","1111111111011110","1111111111011111","1111111111100000","1111111111100001","1111111111100010",
  "","11111111010","1111111111100011","1111111111100100","1111111111100101","1111111111100110","1111111111100111","1111111111101000", "1111111111101001","1111111111101010","1111111111101011",
  "","111111110110","1111111111101100","1111111111101101","1111111111101110","1111111111101111","1111111111110000","1111111111110001","1111111111110010","1111111111110011","1111111111110100",
  "111111110111","1111111111110101","1111111111110110","1111111111110111","1111111111111000","1111111111111001","1111111111111010","1111111111111011","1111111111111100","1111111111111101","1111111111111110"
  };

  int cat = getCat(a);
  lenb = codeLen[n][cat];
  strcpy(b,code[n][cat]);
  int j;
  int c = a;
  if(a<0)
     c+=(int)pow(2,cat)-1;
  for(j=lenb-1;j>lenb-cat-1;j--)
  {
     if(c%2==1)
	b[j] = '1';
     else
	b[j] = '0';
     c/=2;
  }
  b[lenb] = '\0';
}

void Encode(int RL[64], int rl, char* output)
{
//   char output[33*26];
   char b[32];
   int bLen;
   getDCcode(RL[0],bLen,b);
   //   cout<<"Code : "<<RL[0]<<" "<<b;
   strcpy(output,b);
   int i;
   for(i=1;i<rl;i+=2)
   {
	getACcode(RL[i],RL[i+1],bLen,b);
	//	cout<<" , "<<RL[i]<<" "<<RL[i+1]<<" "<<b;
	strcat(output,b);
   }
//   writeToFile(output);
}

void prn(int a[8][8])
{
  for(int i=0;i<8;i++,cout<<"\n")
    for(int j=0;j<8;j++)
      cout<<a[i][j]<<" ";
}

void ZigZag(int QF[8][8],int ZZ[64])
{
  int i=0,j=0,k=0,d=0;
  while(k<36)
    {
      //      cout<<"["<<i<<","<<j<<"]";
      ZZ[k++] = QF[i][j];
      if((i==0)&&(j%2==0))
	{
	  j++;
	  d=1;
	}
      else if((j==0)&&(i%2==1))
	{
	  i++;
	  d=0;
	}
      else if(d==0)
	{
	  i--;
	  j++;
	}
      else
	{
	  i++;
	  j--;
	}
    }
  i = 7;
  j = 1;
  d = 0;
  while(k<64)
    {
      //      cout<<"["<<i<<","<<j<<"]";
      ZZ[k++] = QF[i][j];
      if((i==7)&&(j%2==0))
	{
	  j++;
	  d=0;
	}
      else if((j==7)&&(i%2==1))
	{
	  i++;
	  d=1;
	}
      else if(d==0)
	{
	  i--;
	  j++;
	}
      else
	{
	  i++;
	  j--;
	}
    }
}

void* pth_encode(void* pth_data)
{
  thread_data* my_data = (thread_data*)pth_data;
  int i,j,i1,j1;
  int QF[8][8];
  int **coef = my_data->coef;
  int maxY = my_data->row;
  int maxX = my_data->col;
  //my_data->writecode[0]='\0';
  // cout<<"See me"<<maxY<<"x"<<maxX;
  for(i=0;i<maxY/8;i++)
    for(j=0;j<maxX/8;j++)
    {	
      
      for(i1=0;i1<8;i1++)
	for(j1=0;j1<8;j1++){
	  //cout<<"\n Thread "<<my_data->order<<"i: "<<i<<" j: "<<j<<"|"<<i*8+i1<<"|"<<j*8+j1<<"|"<<maxY<<"x"<<maxX;
	  QF[i1][j1] = coef[i*8+i1][j*8+j1];
	}
      
      	int ZZ[64];
	ZigZag(QF,ZZ);
	
	int RL[64];
	int rl;
	rl = RLE(ZZ,RL);
	
	char  blk_code[1000];
	Encode(RL, rl,blk_code);
	
	strcat(my_data->writecode, blk_code);
    }
  
	
}




