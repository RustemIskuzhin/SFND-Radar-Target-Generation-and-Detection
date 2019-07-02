# SFND Radar Target Generation and Detection
## [Rubric](https://review.udacity.com/#!/rubrics/2548/view) Points
---
#### 1. FMCW Waveform Design
Using the given system requirements, design a FMCW waveform. Find its Bandwidth (B), chirp time (Tchirp) and slope of the chirp.

```Matlab
%% Radar Specifications 
%%%%%%%%%%%%%%%%%%%%%%%%%%%
% Frequency of operation = 77GHz
% Max Range = 200m
range_max = 200
% Range Resolution = 1 m
delta_r = 1
% Max Velocity = 100 m/s
%%%%%%%%%%%%%%%%%%%%%%%%%%%

c = 3*10^8
pos = 110
vel = 20
B = c/(2*delta_r)
Tchirp = 5.5*(range_max*2/c)
slope = B/Tchirp
disp(slope)
```

#### 2. Simulation Loop
Simulate Target movement and calculate the beat or mixed signal for every timestamp.

```Matlab
fc= 77e9;             %carrier freq
                                              
Nd=128;                   % #of doppler cells OR #of sent periods % number of chirps

Nr=1024;                  %for length of time OR # of range cells

t=linspace(0,Nd*Tchirp,Nr*Nd); %total time for samples

Tx=zeros(1,length(t)); %transmitted signal
Rx=zeros(1,length(t)); %received signal
Mix = zeros(1,length(t)); %beat signal

r_t=zeros(1,length(t));
td=zeros(1,length(t));


disp(length(t))
 
for i = 1:length(t)         
    r_t(i) = 110 + vel*t(i);
    td(i) = 2*r_t(i)/c; 
    Tx(i) = cos(2*pi*(fc*t(i) + slope*t(i)^2/2));
    Rx(i) = cos(2*pi*(fc*(t(i) - td(i)) + slope*(t(i) - td(i))^2/2));
    Mix(i) = Tx(i) * Rx(i);
end
```

#### 3. Range FFT (1st FFT)

Implement the Range FFT on the Beat or Mixed Signal and plot the result.

```Matlab
signal = reshape(Mix,[Nr, Nd])
signal_fft = fft(signal, Nr)/Nr;
signal_fft = abs(signal_fft);

signal_fft  = signal_fft(1:Nr/2)   
disp(length(signal_fft))

figure ('Name','Range from First FFT')
subplot(2,1,1)
plot(signal_fft) 
axis ([0 200 0 0.5]);
```

#### 4. 2D CFAR
Implement the 2D CFAR process on the output of 2D FFT operation, i.e the Range Doppler Map.

Determine the number of Training cells for each dimension. Similarly, pick the number of guard cells.

```Matlab
Tr = 8;
Td = 4;
Gr = 8;
Gd = 4;
offset=1.35;
```

Slide the cell under test across the complete matrix. Make sure the CUT has margin for Training and Guard cells from the edges.

```Matlab
for i = 1:(Nr/2-(2*Gr+2*Tr+1))
    for j = 1:(Nd-(2*Gd+2*Td+1))
        ...
    end
end
```

For every iteration sum the signal level within all the training cells. To sum convert the value from logarithmic to linear using db2pow function.

```Matlab
noise_level = zeros(1,1);
for x = i:(i+2*Tr+2*Gr) 
    noise_level = [noise_level, db2pow(RDM(x,(j:(j+ 2*Td+2*Gd))))];
end    
sum_cell = sum(noise_level);
noise_level = zeros(1,1);
for x = (i+Tr):(i+Tr+2*Gr) 
    noise_level = [noise_level, db2pow(RDM(x,(j+Td):(j+Td+2*Gd)))];
end    
sum_guard = sum(noise_level);
sum_train = sum_cell - sum_guard;
```

Average thesummed values for all of the training cells used. After averaging convert it back to logarithmic using pow2db.
Further add the offset to it to determine the threshold.

```Matlab
threshold = pow2db(sum_train/Tcell)*offset;
```

Next, compare the signal under CUT against this threshold.
If the CUT level > threshold assign it a value of 1, else equate it to 0.

```Matlab
signal = RDM(i+Tr+Gr, j+Td+Gd);
if (signal < threshold)
    signal = 0;
else
    signal = 1;
end    
CFAR(i+Tr+Gr, j+Td+Gd) = signal;
```

To keep the map size same as it was before CFAR, equate all the non-thresholded cells to 0.

```Matlab
for i = 1:(Nr/2)
    for j = 1:Nd
       if (i > (Tr+Gr))& (i < (Nr/2-(Tr+Gr))) & (j > (Td+Gd)) & (j < (Nd-(Td+Gd)))
           continue
       end
       CFAR(i,j) = 0;
    end
end
```
Selection of Training, Guard cells and offset.

Training, Guard cells and offset are selected by increasing and decreasing to match the image shared in walkthrough.

Tr = 8;
Td = 4;
Gr = 8;
Gd = 4;
offset=1.35;

Result

![result](https://github.com/RustemIskuzhin/SFND-Radar-Target-Generation-and-Detection/blob/master/images/2D_CFAR.png)
