# Pffft-impulse-convolver
Pretty fast impulse convolution
* Pffft and pffastconvolve
* FFTConvolver
 
# vcvfft.hpp
```c++
#pragma once
#include <pffft.h>
#include <pffastconv.h>

/** Real-valued FFT context.
Wrapper for [PFFFT](https://bitbucket.org/jpommier/pffft/)
`length` must be a multiple of 32.
Buffers must be aligned to 16-byte boundaries. new[] and malloc() do this for you.
*/
struct RealFFT {
	PFFFT_Setup* setup;
	int length;

	RealFFT(size_t length) {
		this->length = length;
		setup = pffft_new_setup(length, PFFFT_REAL);
	}

	~RealFFT() {
		pffft_destroy_setup(setup);
	}

	/** Performs the real FFT.
	Input and output must be aligned using the above align*() functions.
	Input is `length` elements. Output is `2*length` elements.
	Output is arbitrarily ordered for performance reasons.
	However, this ordering is consistent, so element-wise multiplication with line up with other results, and the inverse FFT will return a correctly ordered result.
	*/
	void rfftUnordered(const float* input, float* output) {
		pffft_transform(setup, input, output, NULL, PFFFT_FORWARD);
	}

	/** Performs the inverse real FFT.
	Input is `2*length` elements. Output is `length` elements.
	Scaling is such that IRFFT(RFFT(x)) = N*x.
	*/
	void irfftUnordered(const float* input, float* output) {
		pffft_transform(setup, input, output, NULL, PFFFT_BACKWARD);
	}

	/** Slower than the above methods, but returns results in the "canonical" FFT order as follows.
		output[0] = F(0)
		output[1] = F(n/2)
		output[2] = real(F(1))
		output[3] = imag(F(1))
		output[4] = real(F(2))
		output[5] = imag(F(2))
		...
		output[length - 2] = real(F(n/2 - 1))
		output[length - 1] = imag(F(n/2 - 1))
	*/
	void rfft(const float* input, float* output) {
		pffft_transform_ordered(setup, input, output, NULL, PFFFT_FORWARD);
	}

	void irfft(const float* input, float* output) {
		pffft_transform_ordered(setup, input, output, NULL, PFFFT_BACKWARD);
	}

	/** Scales the RFFT so that `scale(IFFT(FFT(x))) = x`.
	*/
	void scale(float* x) {
		float a = 1.f / length;
		for (int i = 0; i < length; i++) {
			x[i] *= a;
		}
	}
};


/** Complex-valued FFT context.
`length` must be a multiple of 16.
*/
struct ComplexFFT {
	PFFFT_Setup* setup;
	int length;

	ComplexFFT(size_t length) {
		this->length = length;
		setup = pffft_new_setup(length, PFFFT_COMPLEX);
	}

	~ComplexFFT() {
		pffft_destroy_setup(setup);
	}

	/** Performs the complex FFT.
	Input and output must be aligned using the above align*() functions.
	Input is `2*length` elements. Output is `2*length` elements.
	*/
	void fftUnordered(const float* input, float* output) {
		pffft_transform(setup, input, output, NULL, PFFFT_FORWARD);
	}

	/** Performs the inverse complex FFT.
	Input is `2*length` elements. Output is `2*length` elements.
	Scaling is such that FFT(IFFT(x)) = N*x.
	*/
	void ifftUnordered(const float* input, float* output) {
		pffft_transform(setup, input, output, NULL, PFFFT_BACKWARD);
	}

	void fft(const float* input, float* output) {
		pffft_transform_ordered(setup, input, output, NULL, PFFFT_FORWARD);
	}

	void ifft(const float* input, float* output) {
		pffft_transform_ordered(setup, input, output, NULL, PFFFT_BACKWARD);
	}

	void scale(float* x) {
		float a = 1.f / length;
		for (int i = 0; i < length; i++) {
			x[2 * i + 0] *= a;
			x[2 * i + 1] *= a;
		}
	}
};


struct Convolution
{
	PFFASTCONV_Setup *setup;
	int blockLen;
	std::vector<float> buffer;	
	std::vector<float> inputbuffer;
	std::vector<float> outputbuffer;
	Convolution(float * filterCoeffs, size_t filterLen, size_t blockSize)
	{
		blockLen = blockSize;
		setup = pffastconv_new_setup(filterCoeffs,filterLen,&blockLen,0);	
		buffer.resize(blockLen);
	}
	~Convolution() {
		if(setup) pffastconv_destroy_setup(setup);
	}
	
	void Process(int inputLen, float * input, float * output)
	{				
		for(size_t i = 0; i < inputLen; i++)
			inputbuffer.push_back(input[i]);
		int x = pffastconv_apply(setup,inputbuffer.data(),inputbuffer.size(),buffer.data(),true);
		if(x > 0) {
			inputbuffer.erase(inputbuffer.begin(), inputbuffer.begin()+x);
		}
		for(size_t i = 0; i < x; i++)
			outputbuffer.push_back(buffer[i]);
		if(outputbuffer.size() >= inputLen)
		{		
			memcpy(output,outputbuffer.data(),inputLen*sizeof(float));
			outputbuffer.erase(outputbuffer.begin(),outputbuffer.begin()+inputLen);
		}
	}
};

struct StereoConvolution
{
	Convolution * conv[2];
	
	StereoConvolution(float * filterLeft, float * filterRight, size_t filterLen, size_t blockSize) {
		conv[0] = new Convolution(filterLeft,filterLen,blockSize);
		conv[1] = new Convolution(filterRight,filterLen,blockSize);		
	}
	~StereoConvolution() {
		if(conv[0]) delete conv[0];
		if(conv[1]) delete conv[1];
	}
	void Process(int size, float ** inputs, float ** outputs)
	{
		conv[0]->Process(size,inputs[0],outputs[0]);
		conv[1]->Process(size,inputs[1],outputs[1]);
	}
};
		
```

# ImpulseConvolver.hpp

```c++
#pragma once
#include "vcvfft.hpp"
struct ImpulseConvolver
{
    fftconvolver::FFTConvolver convolverL,convolverR;

    ImpulseConvolver(const char * impulse_file)    
    {
        //
        sample_vector<float> lexiconL,lexiconR;
        SndFileReaderFloat sndfile(impulse_file);
        
        std::cout << sndfile.channels() << std::endl;
        std::cout << sndfile.frames() << std::endl;
        std::cout << sndfile.samplerate() << std::endl;
        std::vector<float> channelv(sndfile.frames()*sndfile.channels());
        
        sndfile >> channelv;

        lexiconL.resize(sndfile.frames());
        lexiconR.resize(sndfile.frames());

        for(size_t i = 0; i < sndfile.frames(); i++)
        {        
            lexiconL[i] = channelv[i*2];        
            lexiconR[i] = channelv[i*2+1];
        }

        convolverL.init(BufferSize,lexiconL.data(),lexiconL.size());
        convolverR.init(BufferSize,lexiconR.data(),lexiconR.size());
        
    }
    ~ImpulseConvolver()
    {

    }
    void ProcessBlock(size_t framesPerBuffer, float ** in, float ** out)
    {
        convolverL.process(in[0],out[0],framesPerBuffer);
        convolverR.process(in[1],out[1],framesPerBuffer);
    }
    void InplaceProcess(size_t n, float ** buffer)
    {
        ProcessBlock(n,buffer,buffer);
    }
};

struct PffftConvolver
{
    StereoConvolution * conv;
    
    
    PffftConvolver(const char * impulse_file)    
    {
        //
        sample_vector<float> lexiconL,lexiconR;
        SndFileReaderFloat sndfile(impulse_file);
        
        std::cout << sndfile.channels() << std::endl;
        std::cout << sndfile.frames() << std::endl;
        std::cout << sndfile.samplerate() << std::endl;
        std::vector<float> channelv(sndfile.frames()*sndfile.channels());
        
        sndfile >> channelv;

        lexiconL.resize(sndfile.frames());
        lexiconR.resize(sndfile.frames());

        for(size_t i = 0; i < sndfile.frames(); i++)
        {        
            lexiconL[i] = channelv[i*2];        
            lexiconR[i] = channelv[i*2+1];
        }

        //convolverL.init(BufferSize,lexiconL.data(),lexiconL.size());
        //convolverR.init(BufferSize,lexiconR.data(),lexiconR.size());
        conv = new StereoConvolution(lexiconL.data(), lexiconR.data(), lexiconL.size(),BufferSize);        
    }
    ~PffftConvolver()
    {
        if(conv) delete conv;
    }
    void ProcessBlock(size_t framesPerBuffer, float ** in, float ** out)
    {
        conv->Process(framesPerBuffer,in,out);        
    }
    void InplaceProcess(size_t n, float ** buffer)
    {
        ProcessBlock(n,buffer,buffer);
    }
};
```
