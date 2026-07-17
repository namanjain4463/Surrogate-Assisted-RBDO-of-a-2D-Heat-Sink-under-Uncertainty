# Surrogate-Assisted Reliability-Based Design Optimization of a 2D Heat Sink

This course project couples parametric thermal simulation, surrogate modeling, and reliability analysis to study a two-dimensional aluminum heat sink under uncertain operating conditions. A neural-network surrogate replaces repeated ANSYS solves inside a First-Order Reliability Method (FORM) loop, allowing fin thickness to be optimized against a target reliability index.



## Objective

The design variable is fin thickness, $t_f$. Heat flux, $Q$, and ambient temperature, $T_\infty$, are treated as uncertain inputs. The intended optimization problem is

$$
\min_{t_f} M(t_f)
$$

subject to

$$
P\left[T_{\max}(t_f,Q,T_\infty) \leq T_{\mathrm{lim}}\right] \geq 0.99
$$

and the geometric spacing constraint

$$
s(t_f)=\frac{W_{\mathrm{base}}-n_f t_f}{n_f-1}\geq 0.60\ \mathrm{mm}.
$$

The target reliability is represented in FORM by $\beta_{\mathrm{target}}=\Phi^{-1}(0.99)=2.326$.

## Analysis pipeline

```mermaid
flowchart LR
    A["300-point LHS"] --> B["ANSYS thermal model"]
    B --> C["ANN surrogate"]
    C --> D["FORM / HL-RF"]
    D --> E["Brent root search"]
```

1. Generate a centered-discrepancy-optimized Latin Hypercube design over fin thickness, heat flux, and ambient temperature.
2. Evaluate the design points using a parametric 2D steady-state thermal model in ANSYS.
3. Train a standardized multilayer perceptron to predict maximum heat-sink temperature.
4. Transform the uncertain inputs into standard-normal space and locate the FORM design point using the Hasofer-Lind-Rackwitz-Fiessler algorithm.
5. Search for the fin thickness at which the computed reliability index reaches the target value.

## Model definition

| Quantity | Value used |
|---|---:|
| Base width | 80 mm |
| Base height | 2 mm |
| Fin height | 10 mm |
| Number of fins | 50 |
| Aluminum thermal conductivity | 205 W/(m K) |
| Convective heat-transfer coefficient | 25 W/(m² K) |
| Fin-thickness sampling range | 0.10-1.58 mm |
| Heat-flux distribution | Uniform, 25,000-120,000 W/m² |
| Ambient-temperature reliability model | Normal, $\mu=33$ °C and $\sigma=4$ °C |
| Ambient-temperature training range | 25-40 °C |
| Reliability target | 99% FORM approximation, $\beta=2.326$ |

The ANSYS model uses a two-dimensional cross-section with unit depth. Heat flux is applied at the base and convection is applied to the exposed surfaces. The report states that a 0.2 mm quad-dominant mesh was used.

## Surrogate and reliability methods

The surrogate is a scikit-learn `MLPRegressor` with the following structure:

- Inputs: fin thickness, heat flux, and ambient temperature
- Architecture: 3 -> 24 -> 24 -> 1
- Hidden activation: ReLU
- Optimizer: L-BFGS
- Input preprocessing: `StandardScaler`
- Maximum iterations: 10,000
- Random seed: 42

For each candidate thickness, the notebook maps heat flux and ambient temperature into standard-normal space. The HL-RF iteration estimates the Most Probable Point on the limit-state surface

$$
g(\mathbf{u};t_f)=T_{\mathrm{lim}}-\widehat{T}_{\max}(t_f,\mathbf{u})=0.
$$

Finite differences are used to estimate the limit-state gradient. Brent's method then searches for a thickness satisfying $\beta(t_f)=\beta_{\mathrm{target}}$.

## Stored results

These values reproduce the outputs preserved in `FinalProject.ipynb`; they should not be interpreted as a fully validated design.

| Metric | Stored result | Interpretation |
|---|---:|---|
| ANN training $R^2$ | 0.99999 | In-sample score; no held-out test score is stored |
| Allowable temperature used by the notebook | 129.30 °C | Derived inside the optimization rather than fixed independently |
| Reported fin thickness | 1.5431 mm | FORM root before enforcing the stated gap constraint |
| Reliability index | 2.3264 | FORM estimate |
| Unit-depth mass | 2.515 kg/m | Reported as a unit-depth 2D quantity |

The stored deterministic comparison uses maximum heat flux and mean ambient temperature. It reports 143.72 °C for a 0.10 mm deterministic design and 124.57 °C for the 1.5431 mm FORM design.


## Environment

Python dependencies:

```bash
python -m pip install numpy pandas scipy scikit-learn matplotlib jupyter
```

ANSYS Mechanical is required to regenerate the high-fidelity response data. The notebook expects `Data.csv` to contain six non-data header rows and the following ANSYS parameter columns:

| ANSYS column | Meaning |
|---|---|
| `P2` | Fin thickness in mm |
| `P15` | Ambient temperature in °C |
| `P18` | Heat flux in W/m² |
| `P17` | Maximum temperature in °C |

## Reproduction workflow

1. Run the first notebook cell to create `ansys_input_300.csv`.
2. Import the parameter table into the ANSYS parametric model and update all design points.
3. Export the response table as `Data.csv` using the column layout above.
4. Place `Data.csv` beside the notebook.
5. Restore the missing diagnostic definitions or remove the dependent diagnostics.
6. Restart the kernel and execute the notebook top-to-bottom.
7. Rerun the optimization with $t_f\leq1.012$ mm and an independently specified $T_{\mathrm{lim}}$.
